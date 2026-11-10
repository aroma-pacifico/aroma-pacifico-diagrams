# Flows Diagrams

This directory contains the operational sequence workflows and lifecycle state machines for the Aroma Pacífico drone service, pest-to-fumigation pipeline, and event-driven inference.

---

## Drone Fumigation Service: End-to-End Sequence Diagram

- Source diagram: [`drone-fumigation-sequence.drawio`](drone-fumigation-sequence.drawio)
- Vector export: [`drone-fumigation-sequence.svg`](drone-fumigation-sequence.svg)
- Bitmap preview: [`drone-fumigation-sequence.png`](drone-fumigation-sequence.png)

### Operational Overview

The operational sequence diagram models the five chronological phases of the pest detection and fumigation pipeline. It spans sensor ingestion, threshold alarms, pilot assignment, mission package download, autonomous flight execution, offline encrypted buffering, multipart S3 persistence with upload grant enforcement, Bedrock-driven confirmation and verification, agronomist MFA authorization, and case closure.

| Phase | Operational Focus | Key Actors & Components | Source Section |
|---|---|---|---|
| **Phase 1: Pest Detection & Alerts** | Crop node detects out-of-range variables and publishes via MQTT over TLS 1.2 to AWS IoT Core. Topic rules insert telemetry into DynamoDB and enqueue to SQS. Threshold Lambda evaluates readings, sends SMS/push notifications to farmers (< 5 min), opens alert episode, and raises `crop_alert_raised`. Plot switches to Precision Mode. | Crop Node, AWS IoT Core (D09), Threshold Lambda (D10), SNS, Case Service (D08) | `docs/drone-service.md` §3, §4; `docs/architecture.md` §11 |
| **Phase 2: Confirmation Flight & Bedrock Inference** | Agronomist assigns certified pilot. Pilot authenticates with Cognito MFA at Municipal Seat / Support VLAN 102 and downloads mission package via Application API. Drone preloads package, flies autonomously over focus point, and buffers photos/track offline. IoT Core delivers presigned S3 upload grant. Reception Lambda validates manifest and checksums; CloudTrail audits upload; Bedrock Nova Pro evaluates pest signs (conf >= 0.70). Case transitions to `confirmed`. | Contracted Pilot, Ground Station, Drone Kit (X.509), S3 Objects, Bedrock Nova Pro (D10) | `docs/drone-service.md` §5.1, §6, §7; `docs/inference-functions.md` §3 |
| **Phase 3: Fumigation Order & MFA Authorization** | Case reaching `confirmed` triggers Order Service to create draft order. Agronomist reviews order, sets UTC window, assigns pilot, and specifies product batch. Agronomist authorizes order with Cognito MFA (`auth_time` <= 10 min). Human identity invariant is enforced: function roles are denied authorization permissions. | Agronomist (SR2/JR2), Cognito User Pool (D06), Order Service | `docs/drone-service.md` §5.2; ADR-002; ASM-092 |
| **Phase 4: Mission Execution & Persistence** | Pilot downloads fumigation package before takeoff. Autonomous fumigation flight operates inside authorized window. GPS track and flight record are buffered in encrypted storage. Upon link restore, kit requests MQTT upload grant and uploads track and flight record (`completed` or `aborted`). Reception Lambda persists record; order moves to `executed` (flagged if out of window or batch mismatch). | Pilot, Drone Kit Buffer, Application API (D04), S3 Object Storage, Reception Lambda | `docs/drone-service.md` §5.2, §6.2, §10; ASM-146; PD-141 |
| **Phase 5: Verification & Case Closure** | Order execution plans verification mission for same plot and focus. Pilot flies verification pass and uploads photos and track. Reception Lambda persists record; EventBridge routes to SQS. Verification Lambda invokes Bedrock Nova Pro (`active_signs_at_focus = false`). Order transitions to `closed`; Pest Case transitions to `closed`; plot switches from Precision Mode back to Area Mode. If verification fails or object never arrives, case stays open. | Verification Lambda (D10), Bedrock Nova Pro, DynamoDB Records, Case Service | `docs/drone-service.md` §5.1, §8; ADR-002; ASM-141 |

### Object Storage Hierarchy

Amazon S3 objects are stored under customer-managed KMS key `objects` with Block Public Access and 5-year compliance retention:

| Prefix Pattern | Purpose | Access & Integrity Controls |
|---|---|---|
| `farms/{farm_id}/cases/{case_id}/{mission_type}/{mission_id}/` | Missions for pest cases (`confirmation`, `fumigation`, `verification`). | Strict MQTT upload grant; SHA-256 checksum validation. |
| `farms/{farm_id}/campaigns/{campaign_id}/{mission_type}/{mission_id}/` | Missions for review campaigns (`delimitation`, `estimation`). | Planned by campaign schedule; package digest verified. |
| `farms/{farm_id}/extra/{mission_id}/delimitation/` | Extra delimitation flights for plot boundary updates. | Activated by agronomist; generates new polygon version. |
| `quarantine/{yyyy}/{mm}/{dd}/{original_key}` | Isolation for ungranted keys, revoked pilots/kits, or privacy screens. | Write-only quarantine; excluded from orders and inference. |

---

## Drone Service State Machines: Pest Case, Fumigation Order & Mission Lifecycles

- Source diagram: [`drone-state-machines.drawio`](drone-state-machines.drawio)
- Vector export: [`drone-state-machines.svg`](drone-state-machines.svg)
- Bitmap preview: [`drone-state-machines.png`](drone-state-machines.png)

### State Machine Specifications

The state machine diagram details the formal transitions, guards, triggers, and authorized actors for the three core records defined in Section 5 of `docs/drone-service.md`.

| Lifecycle Domain | Scope & Behavior | Key States & Invariants | Source Section |
|---|---|---|---|
| **5.1 Pest Case Lifecycle** | Tracks pest alert episodes on coffee plots from initial sensor detection to terminal resolution. Plot enters Precision Mode on `opened` and returns to Area Mode only on terminal closure. Duplicate alerts attach to existing case `alert_refs`. | `opened`, `awaiting_delimitation`, `confirming`, `confirmed`, `not_confirmed`, `closed_without_fumigation` (terminal), `in_fumigation`, `verifying`, `closed` (terminal success), `verification_not_closable`. Invariant: If verification object never arrives, case remains in `verifying` and alarms fire. | `docs/drone-service.md` §5.1; ADR-002; ASM-141 |
| **5.2 Fumigation Order Lifecycle** | Controls transactional authorization, dispatch, flight execution, and closure of chemical fumigation. Requires active plot polygon and closed confirmation flight record before creation. | `draft`, `authorized`, `dispatched`, `flown`, `executed`, `closed` (terminal success), `expired` (terminal / recoverable). Invariants: Human authorization mandatory (MFA <= 10 min); no expiration in `flown` while buffered without signal; late flight records in window recover expired orders. | `docs/drone-service.md` §5.2; ADR-002; ASM-142; PD-141 |
| **5.3 Drone Mission Lifecycle** | Manages flight execution across all five mandatory service types (`delimitation`, `confirmation`, `fumigation`, `verification`, `estimation`). Enforces offline store-and-forward and upload grant isolation. | `planned`, `assigned`, `dispatched`, `flown`, `record_persisted` (terminal success), `cancelled` (terminal abort). Invariants: Preload before takeoff; buffer eviction only on persistence receipt; inference triggers on DynamoDB Stream, never on raw S3 PUT. | `docs/drone-service.md` §5.3; ASM-143; ASM-149; PD-104 |

### Cross-Cutting Governance and Roles

| Governance Domain | Role & Constraints | Implementation Standard |
|---|---|---|
| **Agronomist Authority** | Exclusive authority to authorize fumigation orders. Must present fresh MFA session (`auth_time` <= 10 min). Authorizer ID captured from token; cannot be operator. | Cognito `staff` User Pool, IAM policy checks, ASM-092. |
| **Pilot & Kit Registration** | Pilot verified against Aerocivil registry; drone kit registered as AWS IoT Thing with X.509 certificate. Time-bound MQTT upload grants limit S3 writes to manifest keys. | AWS IoT Core Thing registry, TLS 1.2 mTLS, ASM-020, PD-104. |
| **Event-Driven Inference** | Asynchronous Bedrock Nova Pro evaluation triggered via DynamoDB Streams and EventBridge. TransactWriteItems appends confidence scores and outcome. Zero mutation on failure. | Amazon Bedrock Runtime `Converse`, EventBridge Pipes, ASM-160. |
| **Traceability & Audit** | DynamoDB `records` table encrypted under KMS key `records`. S3 data events audited in CloudTrail. Mandatory flight closure (`completed` / `aborted`) in all records. | AWS CloudTrail, KMS customer-managed keys, ASM-145, ASM-146. |

---

### Source Documents

- `docs/drone-service.md`: Services table (§3), records (§4), state machines (§5), mission package and flight record schemas (§6), S3 prefixes and quarantine (§7), inference triggers (§8), ingress endpoints (§9), offline store-and-forward behavior (§10), and review campaign cadence (§11).
- `docs/inference-functions.md`: Bedrock Nova Pro foundation model integration in `us-east-1`, four triggers, function contracts, and confidence thresholds (§2-§5, §8).
- `docs/architecture.md`: Section 11 (pest-to-fumigation flow) and Section 14 (IaC boxes catalog).
- `docs/adr/adr-002-drone-flight-mode-and-governance.md`: Combined flight mode, states, and switching events.
- `docs/decision-log.md`: ASM-020 to ASM-029, ASM-075, ASM-092, ASM-095, ASM-140 to ASM-149, ASM-160 to ASM-169, PD-104, PD-121, PD-141.

### Boxes D01-D13 Shown

- `D04`: Network and Endpoints (S3 interface endpoint in `object_storage` tier, API Gateway in `application` tier, empty `drone_object_ingress` tier ASM-140).
- `D06`: Security and IAM (Cognito `staff` User Pool MFA, customer-managed KMS keys `records`/`objects`/`telemetry`, IoT Thing certificates, separation of duties).
- `D08`: Farm Module (crop sensor node on plot, MQTT over TLS 1.2, 24-hour buffer).
- `D09`: Ingestion (AWS IoT Core topic rules, DynamoDB telemetry table, SQS queue).
- `D10`: Inference Pattern (DynamoDB Stream pipe, EventBridge custom bus, SQS queues, Lambda functions, Bedrock Nova Pro invocation, TransactWriteItems persistence).
