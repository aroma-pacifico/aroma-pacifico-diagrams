# Flows Diagrams

This folder contains the operational workflows, sequence diagrams, and lifecycle state machines for the Aroma Pacífico drone service, pest-to-fumigation pipeline, and event-driven inference.

---

## Drone Fumigation Service: End-to-End Sequence Diagram

- Source diagram: [`drone-fumigation-sequence.drawio`](drone-fumigation-sequence.drawio)
- Vector export: [`drone-fumigation-sequence.svg`](drone-fumigation-sequence.svg)
- Bitmap preview: [`drone-fumigation-sequence.png`](drone-fumigation-sequence.png)

### What it shows

1. **Pest Detection and Crop Alert (Phase 1)**:
   - Crop node on farm plot detects out-of-range NPK, moisture, or temperature variables.
   - Node publishes reading via MQTT 3.1.1/5 over TLS 1.2 or later (port 8883 / 443 ALPN, mutual X.509 authentication) to the regional AWS IoT Core endpoint in `us-east-1` (or drains oldest-first from its 24-hour non-volatile encrypted buffer upon signal restore, ASM-074).
   - AWS IoT Core topic rule executes two decoupled actions:
     - Action 1: PutItem into the DynamoDB `telemetry` table under the customer-managed `telemetry` KMS key (5-year retention).
     - Action 2: Enqueues the reading key and payload to the Amazon SQS threshold queue (D09 Telemetry Ingestion pattern module, ASM-161).
   - Threshold Lambda function evaluates the reading against the Federation threshold version in force at the reading timestamp (D10). Opens an alert episode, performs episode deduplication, and publishes an alert to the Amazon SNS alert topic (delivering SMS and push notifications to the farmer within the under 5-minute design target, ASM-075). Post-alert: invokes Amazon Bedrock for plausibility evaluation on episode openings without blocking notifications (ASM-167).
   - Threshold Lambda persists `crop_alert_raised` event into the DynamoDB `records` table (ASM-141).
   - Platform Case Service switches the affected plot from Area Mode to Precision Mode (ADR-002). If the farm lacks an active plot polygon, the case enters `awaiting_delimitation` (ASM-027); if an active polygon version exists, the case enters `confirming` and plans a `confirmation` mission with the crop node coordinates as the focus point (ASM-026, ASM-148).

2. **Confirmation Mission, Focus Flight, S3 Persistence and Bedrock Crop Inference (Phase 2)**:
   - Agronomist assigns a contracted certified pilot (group `drone_operator`, valid Aerocivil permit expiry past the mission window, ASM-020, ASM-148).
   - Pilot signs in with multi-factor authentication (MFA) via Amazon Cognito `staff` user pool at a municipal seat or Collection Center Support VLAN 102 (10.4x.2.10, A11 sec 4.1).
   - Pilot downloads the confirmation mission package via the Application API front door / `execute-api` private interface endpoint. The package carries WGS 84 GeoJSON plot boundaries, no-fly zones, focus coordinates, and a platform-computed SHA-256 `package_digest` (ASM-144). The mission transitions to `dispatched`.
   - Pilot preloads the mission package into the drone flight controller before takeoff ("La misión se carga antes del despegue").
   - Drone executes an autonomous precision flight over the focus point. If signal is unavailable over the plot, focus photos, GPS track, and flight records are stored in the local encrypted non-volatile buffer (ASM-095). The mission remains `dispatched` and the case remains open.
   - Link restored: Drone kit connects to AWS IoT Core, reports `flight_completed` status with package digest, and requests an upload grant via MQTT topic. IoT Core delivers the grant, restricting S3 uploads strictly to the mission's manifest keys (A09 sec 6, PD-104).
   - Ground station uploads focus photos and flight record via HTTPS multipart upload to Amazon S3:
     - `farms/{farm_id}/cases/{case_id}/confirmation/{mission_id}/photos/{seq}.jpg`
     - `farms/{farm_id}/cases/{case_id}/confirmation/{mission_id}/record/flight-record.json`
     (Using S3 interface endpoint in `object_storage` via IPsec VPN tunnel, or public regional S3 endpoint).
   - Upload is audited by AWS CloudTrail S3 data events (`PutObject`, `CompleteMultipartUpload`).
   - Reception Lambda function (not VPC-attached, ASM-140) validates manifest checksums, SHA-256 package digest, and grant validity. Files outside the grant are quarantined to `quarantine/{yyyy}/{mm}/{dd}/` (PD-104). Persists `record_persisted` to the DynamoDB `mission` table, and returns a persistence receipt. Ground station deletes local buffer objects only after confirming receipt (A10 sec 6).
   - DynamoDB Stream on the mission table feeds an EventBridge Pipe that puts a `mission_record_persisted` event on the custom event bus, which routes via rule to the SQS confirmation queue (ASM-149, ASM-161).
   - Confirmation Lambda reads the SQS queue, fetches persisted focus photos from S3 (verifying `person_screen = clear`, PD-015), and calls Amazon Bedrock Runtime `Converse` operation (Amazon Nova Pro foundation model in `us-east-1`, zero-shot prompt, ASM-160, ASM-168).
   - Confirmation Lambda executes a DynamoDB `TransactWriteItems`: appends `confirmation_result` (outcome `focus_confirmed`, timestamp, model confidence >= 0.70 threshold, source S3 key/version) and transitions the Pest Case from `confirming` to `confirmed` (ASM-141). The function role has zero authorization permissions and never authorizes fumigation.

3. **Fumigation Order Creation, Agronomist Review and Human MFA Authorization (Phase 3)**:
   - Case reaching `confirmed` triggers the Platform Order Service to create a draft Fumigation Order (`farm_id`, `plot_id`, `reason`: `case_id` + `focus_confirmed` result, `polygon_version`). Guard: active plot polygon and closed confirmation record (ASM-142). State: `draft`.
   - Agronomist reviews the draft order, sets UTC window (`window_start`, `window_end`), assigns pilot (`operator_id`), and specifies the planned `product_batch_id` (ADR-002, ASM-029).
   - Agronomist authorizes the order: Caller must be in group `agronomist`, scoped to the farm's department, with a fresh Cognito MFA session (`auth_time` at most 10 minutes old, ASM-092).
   - `authorizer_id` is captured strictly from the token payload; function roles are denied (A09 sec 4.3). Order transitions from `draft` to `authorized`. Human authorization is mandatory: *"La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta."*

4. **Mission Preload, Fumigation Flight, Encrypted Buffering and Record Execution (Phase 4)**:
   - Assigned pilot signs in with MFA, downloads the fumigation mission package via API (`window`, `polygon_version`, no-fly zones, planned `product_batch_id`, authorizer reference, SHA-256 digest).
   - Application API transitions Fumigation Order and Mission to `dispatched`.
   - Pilot preloads the fumigation package into the drone kit before takeoff.
   - Pilot and drone fly fumigation over the plot inside the authorized window. (Dose and chemical mixture are out of scope per case statement).
   - Flight record, closure status (`completed` or `aborted`), and GPS track are written to local encrypted storage.
   - Link restored: kit reports `flight_completed` status (moving order to `flown`) and requests an upload grant via MQTT.
   - Ground station uploads track and flight record via HTTPS multipart upload to S3:
     - `farms/{farm_id}/cases/{case_id}/fumigation/{mission_id}/track/{seq}.json`
     - `farms/{farm_id}/cases/{case_id}/fumigation/{mission_id}/record/flight-record.json`
     (Includes `pilot_subject_id`, `kit_thing_name`, `product_batch_id_used`, takeoff/landing times, and closure reason).
   - Reception Lambda validates manifest, grant, and checksums. Persists flight record into DynamoDB `records`. Returns receipt; buffer evicted.
   - Fumigation Order transitions from `flown` to `executed` (ASM-146). If flown outside window, batch mismatch, or aborted closure, the order is flagged rather than blocked (ASM-142, ASM-146). Late in-window records recover expired orders (`late_record` flag).
   - Store-and-Forward Invariant: An order in `flown` does not expire at `window_end` while buffered without signal (PD-141).

5. **Post-Fumigation Verification Flight, Bedrock Evaluation and Case Closure (Phase 5)**:
   - Order reaching `executed` triggers the platform to plan a `verification` mission for the same plot and focus point (ASM-141). Pest Case transitions from `in_fumigation` to `verifying`.
   - Assigned pilot downloads verification mission package and flies a short pass over the focus point.
   - Ground station uploads verification photos and flight record:
     - `farms/{farm_id}/cases/{case_id}/verification/{mission_id}/photos/{seq}.jpg`
     - `farms/{farm_id}/cases/{case_id}/verification/{mission_id}/record/flight-record.json`
   - Reception Lambda validates manifest, checksums, and grant; persists `record_persisted` for the verification mission.
   - DynamoDB Stream pipe to EventBridge custom bus routes event to the SQS verification queue (ASM-149).
   - Verification Lambda reads SQS, reads S3 verification photos, checks `person_screen = clear`, and invokes Amazon Bedrock Nova Pro.
   - Verification Lambda executes DynamoDB `TransactWriteItems`: writes `verification_results` (`can_close = true`), transitions Fumigation Order from `executed` to `closed`, and transitions Pest Case from `verifying` to `closed` (`case_closed` event) (ASM-141, Part E).
   - Case closure triggers plot state switch: Plot returns from Precision Mode back to Area Mode (ADR-002).
   - Closure Invariant: If the verification photo or track does not arrive, or the function fails, state does not mutate: the case stays open in `verifying`, and CloudWatch alarms report unclosed fumigations (Point 2 / A15 sec 5.1 / ASM-141 / PD-141). *"Si ese archivo no llega, la fumigación sigue abierta."*

6. **Amazon S3 Drone Object Bucket Hierarchy and Governance Legend**:
   - Bucket in workload account (`pdn` / `us-east-1`), encrypted under customer-managed KMS key `objects`, Block Public Access enabled, versioned, 5-year compliance retention (ASM-145).
   - Key prefixes:
     - Case missions: `farms/{farm_id}/cases/{case_id}/{mission_type}/{mission_id}/` (types: `confirmation`, `fumigation`, `verification`).
     - Sub-prefixes: `package/mission-package-v{n}.json`, `record/flight-record.json`, `photos/{seq}.jpg`, `track/{seq}.json`, `maps/`, `polygons/`.
     - Campaign missions: `farms/{farm_id}/campaigns/{campaign_id}/{mission_type}/{mission_id}/` (types: `delimitation`, `estimation`).
     - Extra delimitation: `farms/{farm_id}/extra/{mission_id}/delimitation/`.
     - Quarantine prefix: `quarantine/{yyyy}/{mm}/{dd}/{original_key}` (unmatched grants, revoked credentials, or person screen quarantine; never linked to orders or read by inference, PD-104).

### Source documents

- `docs/drone-service.md` (all sections): Services table (sec 3), records (sec 4), state machines (sec 5), mission package and flight record schemas (sec 6), S3 prefixes and quarantine (sec 7), inference triggers (sec 8), ingress endpoints (sec 9), offline store-and-forward behavior (sec 10), and review campaign cadence (sec 11).
- `docs/inference-functions.md` (sections 2, 3, 4, 5, 8): Managed inference using Amazon Bedrock Nova Pro in `us-east-1`, four triggers, function contracts, result schemas, and person screening (PD-015).
- `docs/architecture.md` (section 11): Numbered pest-to-fumigation operational flow.
- `docs/case/case-statement.pdf`: Epic 4, Part B, Part E, points 1, 2, 6.
- `docs/decision-log.md`: ADR-002, ASM-020 to ASM-029, ASM-075, ASM-092, ASM-095, ASM-140 to ASM-149, ASM-160 to ASM-169, PD-104, PD-121, PD-141.

### Boxes D01-D13 shown

- `D04`: VPC and tiers of pattern environment (S3 interface endpoint in `object_storage` tier, API Gateway in `application` tier, empty `drone_object_ingress` tier ASM-140).
- `D06`: Identities, KMS customer-managed keys (`records`, `objects`, `telemetry`), Cognito User Pools (`staff` MFA), IoT Core Thing certificates, and separation of duties.
- `D08`: Reusable farm module (crop sensor node on plot, MQTT over TLS 1.2, 24-hour buffer).
- `D09`: Telemetry ingestion pattern module (AWS IoT Core topic rule, DynamoDB telemetry table, SQS queue).
- `D10`: Event-driven inference function pattern (DynamoDB Stream pipe, EventBridge custom bus and rules, SQS queues, Lambda functions, Bedrock Nova Pro invocation, DynamoDB TransactWriteItems persistence).

---

## Drone Service State Machines: Pest Case, Fumigation Order & Mission Lifecycles

- Source diagram: [`drone-state-machines.drawio`](drone-state-machines.drawio)
- Vector export: [`drone-state-machines.svg`](drone-state-machines.svg)
- Bitmap preview: [`drone-state-machines.png`](drone-state-machines.png)

### What it shows

1. **Pest Case Lifecycle (Section 5.1)**:
   - Complete state machine across 10 states: `opened`, `awaiting_delimitation`, `confirming`, `confirmed`, `not_confirmed`, `closed_without_fumigation` (terminal), `in_fumigation`, `verifying`, `closed` (terminal success), and `verification_not_closable`.
   - Mode Boundary: Plot switches from Area Mode to Precision Mode upon case opening (`crop_alert_raised`); returns to Area Mode strictly upon terminal closure (`closed` or `closed_without_fumigation`).
   - Deduplication Guard: Incoming alerts for a plot with an open case join its `alert_refs`; no duplicate case is opened (ASM-141).
   - Polygon Precondition: If the farm lacks an active polygon version, the case waits in `awaiting_delimitation` until a `delimitation` flight activates a new GeoJSON polygon version (ASM-027, ASM-147).
   - Confirmation Flow: Transitions `confirming` to `confirmed` on `focus_confirmed` (Bedrock pest signs present with confidence >= 0.70 threshold) or to `not_confirmed` on `focus_not_confirmed`. Agronomist re-checks allow closing without fumigation or requesting re-confirmation.
   - Closure Invariant: Transitions `verifying` to `closed` on `case_closed` (Bedrock verification result `can_close = true`, objects persisted, order closed). If objects fail to arrive, the case stays in `verifying` and alarms fire (ASM-141, PD-141).

2. **Fumigation Order Lifecycle (Section 5.2)**:
   - Complete state machine across 7 states: `draft`, `authorized`, `dispatched`, `flown`, `executed`, `closed` (terminal success), and `expired` (terminal / recoverable with reasons `window_elapsed`, `rejected`, `pilot_revoked`, `kit_revoked`).
   - Prerequisites for Order Creation: Plot must have an active polygon version, and confirmation flight record must be closed (ASM-142). *"Si no hay polígono del lote o no hay cierre del registro, la orden no sale."*
   - Human Identity Mandatory: Authorization requires group `agronomist`, farm department scope, and fresh MFA (`auth_time` <= 10 min, ASM-092). `authorizer_id` is captured from the token; function roles are strictly denied. *"La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta."*
   - No-Signal Store-and-Forward: An order in `flown` does not expire at `window_end` while buffered without signal (PD-141).
   - Execution Invariants: Any persisted fumigation flight record transitions the order to `executed`. Flags are recorded for `out_of_window`, `batch_mismatch`, `aborted_closure`, or `late_record` (ASM-146).
   - Late Recovery: A late flight record whose flight times occurred within the authorized window transitions an `expired` (`window_elapsed`) order into `executed` (flagged `late_record`).
   - Order Closure: Moves to `closed` on any persisted verification result (both `can_close` and `cannot_close` close the order, ASM-141).

3. **Drone Mission Lifecycle (Section 5.3)**:
   - Complete state machine across 6 states: `planned`, `assigned`, `dispatched`, `flown`, `record_persisted` (terminal success), and `cancelled` (terminal abort).
   - Five Mandatory Services: Covers `delimitation`, `confirmation`, `fumigation`, `verification`, and `estimation`. *"Los cinco entran. No se sustituye uno por otro."*
   - Preload and Buffer Invariant: Mission package must be downloaded before flight. Storage buffer evicts objects only after receiving a checksum-confirmed persistence receipt.
   - Upload Quarantine: Objects failing grant, checksum, or revocation checks go to `quarantine/{yyyy}/{mm}/{dd}/` and do not count toward persistence (PD-104).
   - Decoupled Inference Trigger: Drone inference triggers on `mission_record_persisted` via DynamoDB Streams to EventBridge, never on raw S3 object arrival (ASM-149).
   - Observability: Missions in `dispatched` or `flown` without a persisted record are tracked by CloudWatch alarms as unpersisted flights (Point 6 / A21).

4. **Cross-Cutting Governance, Actor Roles and Invariants**:
   - Detailed specification cards covering the Agronomist Role (MFA <= 10 min, authorizer), Contracted Pilot & Drone Kit (mTLS X.509, upload grants, encrypted buffers), Event-Driven Inference (Amazon Bedrock Nova Pro, zero-shot prompts, failure safety), and Case/Order/S3 Traceability (KMS keys, 5-year retention, CloudTrail data events audit, opaque identifiers).

### Source documents

- `docs/drone-service.md` (Section 5.1 Pest Case, Section 5.2 Fumigation Order, Section 5.3 Mission): Authoritative state transition tables, guards, triggers, and actors.
- `docs/inference-functions.md` (sections 3, 4, 5, 8): Bedrock inference integration, confidence scoring, and result schemas.
- `docs/architecture.md` (section 11, section 14): Pest-to-fumigation flow and IaC box traceability.
- `docs/adr/adr-002-drone-flight-mode-and-governance.md`: Combined flight mode, states, and switching events.
- `docs/decision-log.md`: ASM-020 to ASM-029, ASM-092, ASM-140 to ASM-149, ASM-160 to ASM-169, PD-104, PD-141.

### Boxes D01-D13 shown

- `D06`: Identities (Cognito `staff` User Pool MFA, IoT Thing certificates), customer-managed KMS key `records`, and separation of duties.
- `D09`: Telemetry ingestion trigger on crop telemetry readings.
- `D10`: Event-driven inference function pattern for confirmation and verification evaluation and DynamoDB transactional state updates.
