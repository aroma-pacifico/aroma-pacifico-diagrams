# Flows Diagrams

This directory contains architectural and operational sequence diagrams and lifecycle state machines for event-driven workflows, data flows, automated alert pipelines, and drone service execution in the Aroma Pacífico architecture.

---

## Farm Monitoring and Pilot Alerts Sequence

- Source diagram: [`farm-monitoring-sequence.drawio`](farm-monitoring-sequence.drawio)
- Vector export: [`farm-monitoring-sequence.svg`](farm-monitoring-sequence.svg)
- Bitmap preview: [`farm-monitoring-sequence.png`](farm-monitoring-sequence.png)

### Architectural Overview

The sequence diagram documents the end-to-end event flow from initial field measurement at coffee farms in Cauca, Huila, and Tolima to multi-channel alert delivery on the farmer mobile device. The architecture handles both routine crop monitoring (6 readings per day across 200 pilot farms) and continuous harvest drying monitoring (24 readings per day across active drying lots), scaling cleanly to 1,000 farms without requiring architectural redesign.

Field nodes capture readings and append them to an encrypted non-volatile local ring buffer. The node wakes its cellular radio (LTE-M, NB-IoT, or Cat-1, with satellite fallback for remote locations) and establishes a mutual TLS 1.2+ connection with AWS IoT Core in `us-east-1` using a unique X.509 client certificate. Upon receiving an MQTT `PUBACK` (QoS 1), the local record is evicted from the ring buffer.

AWS IoT Core topic rules decouple ingestion from processing through dual parallel actions: an immutable write to the DynamoDB `telemetry` table under customer-managed key `telemetry` (fulfilling A08 audit requirements with 5-year TTL retention), and an asynchronous message enqueue to the SQS threshold queue. The Threshold Lambda processes batches from SQS, performs a strongly consistent read from DynamoDB to ensure write ordering, and queries active Federation threshold versions from the DynamoDB `thresholds` table (KMS `records`) effective at `original_timestamp`.

### Latency Budget Allocation

The architecture enforces a strict 7-hop latency budget derived from `docs/connectivity.md` section 8.2 and `docs/farm-monitoring.md` section 7.2. The sum of per-hop target latencies is 210 seconds (3.5 minutes), leaving a 90-second safety margin for cellular retries and network jitter within the 300-second (5-minute) SLA limit defined in the project case statement.

| Hop | Segment | Target Latency | Architectural Responsibilities |
|-----|---------|----------------|--------------------------------|
| 1 | Node Capture | 10 s | Sensor measurement taken and assembled into payload. Nodes perform zero threshold evaluation logic. |
| 2 | Radio & TLS | 60 s | Radio wake, cellular network attach (LTE-M/NB-IoT/Cat-1 or satellite), and mutual TLS 1.2+ handshake on port 8883/443 ALPN. |
| 3 | MQTT Publish | 20 s | MQTT QoS 1 publish to AWS IoT Core regional endpoint; local buffer released only upon `PUBACK`. |
| 4 | Ingestion Routing | 10 s | Topic rule executes dual actions: writes telemetry to DynamoDB and enqueues message key to SQS threshold queue with DLQ. |
| 5 | Rules & Episodes | 20 s | Threshold Lambda reads telemetry, queries Federation thresholds table at `original_timestamp` (zero cache), and deduplicates episodes. |
| 6 | PII & Coalescing | 30 s | Dispatcher Lambda resolves farmer contact information from DynamoDB `personal-data` (KMS `personal`) and applies SMS coalescing. |
| 7 | Multi-Channel Dispatch | 60 s | Parallel transmission via AWS End User Messaging SMS and Amazon SNS Mobile Push (FCM / APNs) to farmer phone. |
| **Total** | **Target Sum** | **210 s** | **90 s retry and jitter margin reserves remaining time within 300 s (< 5 min) case SLA limit.** |

### Store-and-Forward and Offline Drain

During cellular signal loss or backhaul outages, field nodes buffer readings in non-volatile encrypted storage for up to 24 hours. Any unacknowledged readings evicted after 24 hours are counted and reported to cloud telemetry via a `telemetry_gap` record.

Upon network reconnection, the node transmits the latest live reading first to provide immediate field status, followed by buffered readings drained in chronological order (oldest first). Each drained reading is evaluated against Federation thresholds active at its `original_timestamp`. Readings where `arrival_time - original_timestamp > 300 s` are flagged with `delayed = true`, enabling farmers to distinguish immediate conditions from historical alerts while deduplication guarantees at most one alert notification per drainage burst.

### Federation Thresholds and Episode Deduplication

Thresholds are treated as authoritative Federation configuration data stored in the DynamoDB `thresholds` table (KMS `records`), never hardcoded in application logic. Initial baseline ranges span T-01 through T-10, encompassing soil pH, ambient and soil moisture, ambient and soil temperature, and grain drying metrics. Farm-specific overrides supersede pilot-wide defaults, and zero in-memory caching ensures that configuration updates apply immediately to subsequent readings. Full traceability is maintained through CloudTrail data events and DynamoDB Streams archiving to S3 Object Lock.

To prevent alert fatigue across more than 100,000 monthly readings, the system manages alerts using an episode lifecycle model. The first out-of-range reading opens an episode via conditional put in the DynamoDB Alert Store and emits a `crop_alert_raised` event (triggering ADR-002 drone confirmation workflow). Subsequent out-of-range readings attach to the open episode to update counts and extreme values without dispatching redundant notifications. When values return to acceptable ranges, the episode is closed with an `alert_resolved` push notification.

### Multi-Channel Notification and PII Isolation

Farmer notifications follow a dual-channel strategy tailored to rural field connectivity (PD-085, ASM-136). Plain-text SMS delivers immediate, reliable notification across basic cellular coverage without requiring mobile data balances. Push notifications deliver detailed history curves and status cards to the mobile application when data coverage is available. To reduce noise, an SMS coalescing window aggregates multiple alerts for the same farm within 10 minutes into a single follow-up message.

Strict PII isolation is enforced across all processing boundaries. Ingestion and evaluation components handle only technical telemetry and farm aliases under KMS key `telemetry`. The Notification Dispatcher Lambda performs identity resolution against DynamoDB `personal-data` under customer-managed key `personal` and never accesses raw sensor readings. SMS messages contain no personal names, national identification numbers, or clickable URLs. Carrier delivery receipts and mobile push receipts are recorded in the Alert Store to serve as service level indicators (SLIs) for the alert-delivery SLO.

---

### Source Documents

- `aroma-pacifico-architecture/docs/farm-monitoring.md`:
  - Section 1: Purpose and requirements
  - Section 3: Harvest load profile, 5-year volume projections, and 1,000 farms scale
  - Section 4: Federation thresholds data model, initial ranges T-01..T-10, and change audit
  - Section 5: Step-by-step telemetry classification and persistence
  - Section 6: Episode deduplication lifecycle, buffer drainage, and drying lot management
  - Section 7: Dual-channel push and SMS strategy, dispatch path, and latency budget
  - Section 8: Farmer perception, mobile application views, and sample SMS formats
  - Section 9: Historical reporting and pilot analytics
  - Section 10: Observability hooks and delivery receipt tracking
- `aroma-pacifico-architecture/docs/connectivity.md`:
  - Section 4.2 & 4.3: Exit protocol, mutual TLS, 24-hour buffer, and store-and-forward drain
  - Section 4.4: Out-of-coverage farms and satellite IP bearer support
  - Section 8: 5-minute alert latency budget and tie to alert-delivery SLO
  - Section 11: Cellular network generation standards (LTE-M, NB-IoT, Cat-1, zero 2G/3G dependency)
- `aroma-pacifico-architecture/docs/architecture.md`:
  - Section 11: Pest-to-fumigation integration and `crop_alert_raised` trigger
  - Section 12: Store-and-forward and offline behavior matrix
  - Section 14: Boxes D01-D13 catalog and IaC traceability mapping
- `aroma-pacifico-architecture/docs/decision-log.md`:
  - `ASM-053`, `ASM-054`, `ASM-059`: Soil pH, NPK publication status, and drying closure criteria
  - `ASM-067`: 5-year telemetry retention under S3 Object Lock
  - `ASM-070`: Dual node architecture per farm with dedicated cryptographic identities
  - `ASM-074`: 24-hour non-volatile ring buffer, live-first drain, and gap reporting
  - `ASM-075`: Per-hop latency budget targets
  - `ASM-091`, `ASM-092`: Passwordless SMS authentication for field personnel
  - `ASM-097`: KMS key segregation (`telemetry`, `records`, `personal`)
  - `ASM-130` - `ASM-138`: Drying lot controls, separate air/soil limits, episode deduplication, and timestamp evaluation
  - `ASM-161`: SQS threshold queue decoupling and strongly consistent telemetry read
  - `ASM-162`: Lambda failure handling and concurrency parameters
  - `ASM-167`: Asynchronous Bedrock Nova Pro plausibility check on opening readings
  - `ADR-002`: Combined drone mode switch on `crop_alert_raised`
  - `ADR-003`: IaC pattern module, IoT Core ingestion, and SNS alert topic
  - `PD-074`: DynamoDB threshold configuration store and audit architecture
  - `PD-085`: Push and SMS dual-channel alert decision
  - `PD-130`, `PD-134`, `PD-135`: Service quotas, delivery receipt SLIs, and threshold evaluation contracts

---

### Diagram Boxes D01-D13 Shown

- **`D08`** (Track B issue #28 / B08): Reusable Farm Module (field node, crop and drying sensors, 24h buffer, device X.509 certificate).
- **`D09`** (Track B issue #29 / B09): Telemetry Ingestion Pattern Module (AWS IoT Core, topic rules, DynamoDB telemetry table, error SQS queue).
- **`D10`** (Track B issue #30 / B10): Event-Driven Inference & Function Pattern (SQS threshold queue, Lambda rules engine, DynamoDB Alert Store, SNS Alert Topic).
- **`D06`** (Track B issue #26 / B06): Identity & Access Controls (device X.509 certificates, IAM execution roles, KMS customer-managed encryption keys `telemetry`, `records`, `personal`).

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
- `docs/adr/adr-004-drone-capability-orchestration.md`: the bus and its routing rules are grouped and labeled the Drone capability orchestrator (event-driven).
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

---

## Sack State Machine and Lifecycle

- Source diagram: [`sack-states.drawio`](sack-states.drawio)
- Vector export: [`sack-states.svg`](sack-states.svg)
- Bitmap preview: [`sack-states.png`](sack-states.png)

### What it shows

This diagram models the complete lifecycle of a coffee sack across the seven states specified in Part C of the case statement and Design System badges (`in_preparation`, `in_transit_to_center`, `at_center_accepted`, `at_center_rejected`, `in_transit_to_port`, `delivered_at_port_accepted`, and `delivered_at_port_rejected`). It captures operational transitions T1 through T8 with their triggering events, responsible actors (Farmer, Transporter, and Collection-Center Operator), and guard conditions, alongside the offline receiver-authoritative reconciliation rule (ASM-152) and the five non-state-changing events (ASM-151). The lower panels detail the append-only DynamoDB event store topology under customer-managed KMS keys (`records`, `telemetry`, `public_view`), the deterministic history replay fold, and disaster recovery objectives (RPO 15 minutes, RTO 4 hours).

### Source documents

| Source Document | Sections | Covered Topics and Decisions |
|---|---|---|
| `aroma-pacifico-architecture/docs/sack-and-trip.md` | Sec 1–4, 6, 8–10 | Sack lifecycle, transition table T1–T8, offline fold rules, event schema, center reserve (>= 400 codes), store topology, RPO 15 min / RTO 4 h |
| `aroma-pacifico-architecture/docs/security-and-identity.md` | Sec 3.2, 4.1–4.2, 7.2, 8 | Cognito `staff` (MFA) and `field` user pools, KMS keys (`records`, `telemetry`, `public_view`), role catalog |
| `aroma-pacifico-architecture/docs/architecture.md` | Sec 14 | Box catalog D01–D13 and IaC module traceability |
| `aroma-pacifico-architecture/docs/decision-log.md` | Decision entries | ASM-007, ASM-057, ASM-076, ASM-077, ASM-079, ASM-091, ASM-094, ASM-096, ASM-150–159, PD-096, PD-110, PD-151, PD-153, PD-154 |

### Boxes D01–D13 shown

| Box | Catalog Name (`architecture.md` sec 14) | Architectural Scope in Diagram |
|---|---|---|
| `D04` | VPC and tiers of the pattern environment | VPC private subnets and routing boundaries enclosing the application and data tiers |
| `D05` | Association of the environment with one pattern collection center | Physical collection center association (Cauca, Huila, Tolima receiving posts, LAN/printer links) |
| `D06` | Identities, KMS keys and initial access controls | Customer-managed KMS keys (`records`, `telemetry`, `public_view`), IAM execution roles, and Cognito user pools |

*Components belonging to no box*: The DynamoDB transactional tables (`sack_event_log`, `sacks`, `sack_history`, `trips`, `transport_positions`, `buyer_view`) and application worker functions are application workload components and belong to no IaC D-box (marked `-` in `architecture.md` section 14).

---

## Sack, Trip and Buyer Public Lookup Sequence

- Source diagram: [`sack-and-trip-sequence.drawio`](sack-and-trip-sequence.drawio)
- Vector export: [`sack-and-trip-sequence.svg`](sack-and-trip-sequence.svg)
- Bitmap preview: [`sack-and-trip-sequence.png`](sack-and-trip-sequence.png)

### What it shows

This diagram illustrates the end-to-end operational sequence of sack registration, transport custody, collection center reception, and unauthenticated public query across six distinct phases. It details pre-printed code sheet generation at the collection center via IPP over TCP 631, wet-mill sack binding and scale weighing with offline outbox buffering on the Farmer App, 5-minute GNSS tracking during transporter transit across cellular dead zones, center reception with reserve label binding for unlabeled arrivals, and highway transit to Buenaventura port. Phase 6 models the unauthenticated public lookup architecture via Amazon CloudFront, AWS WAF rate-limiting, and Lookup Lambda URL with Origin Access Control (OAC), demonstrating strict isolation of the `buyer_view` DynamoDB projection, anonymous audit logging to `/aws/application/audit`, and the security boundary withholding producer personal data and internal telemetry.

### Source documents

| Source Document | Sections | Covered Topics and Decisions |
|---|---|---|
| `aroma-pacifico-architecture/docs/sack-and-trip.md` | Sec 2–7, 8–10 | Registration at farm, custody handover, 5-min GNSS fixes, code sheets, public lookup, peak load arithmetic, store schema |
| `aroma-pacifico-architecture/docs/security-and-identity.md` | Sec 3.2, 4.2, 7.2, 8.1–8.4, 9.1 | Cognito pools, KMS key separation, Crockford Base32 format, buyer view projection, withheld sensitive fields (cédula, GPS) |
| `aroma-pacifico-architecture/docs/connectivity.md` | Sec 5, 7, 10 | Store-and-forward outbox buffering, priority drain order (custody first, positions oldest first) |
| `aroma-pacifico-architecture/docs/firewall-policy.md` | Rules CL-PUB-01 to CL-PUB-04 | Edge WAF Web ACL inspection, rate limits (100 req / 5 min per IP), CloudFront OAC |
| `aroma-pacifico-architecture/docs/landing-zone.md` | Sec 7.2 | Anonymous lookup audit in `/aws/application/audit` CloudWatch log group |
| `aroma-pacifico-architecture/docs/architecture.md` | Sec 14 | Box catalog D01–D13 and IaC module traceability |
| `aroma-pacifico-architecture/docs/decision-log.md` | Decision entries | ASM-057, ASM-058, ASM-069, ASM-076, ASM-077, ASM-079, ASM-094, ASM-096, ASM-150–159, PD-051, PD-096, PD-102, PD-110, PD-151, PD-153, PD-154 |

### Boxes D01–D13 shown

| Box | Catalog Name (`architecture.md` sec 14) | Architectural Scope in Diagram |
|---|---|---|
| `D04` | VPC and tiers of the pattern environment | Private application tier subnets and endpoints (hosting application APIs and handlers) |
| `D05` | Association of the environment with one pattern collection center | Center receiving post and local network IPP code-sheet printer (TCP 631) |
| `D06` | Identities, KMS keys and initial access controls | Customer-managed KMS keys (`records`, `telemetry`, `public_view`), IAM execution roles, and Cognito pools |
| `D07` | The pattern firewall rules | Cloud firewall controls and AWS WAF web ACL (`CL-PUB-01`) terminating TLS 1.2+ and rate-limiting at edge |

*Components belonging to no box*: The DynamoDB tables (`sack_event_log`, `sacks`, `trips`, `transport_positions`, `buyer_view`), Application Lambdas (`sack-events`, `sack-applier`, `sack-exporter`), CloudWatch Logs audit log group (`/aws/application/audit`), and public lookup components (CloudFront distribution, Lookup Lambda Function URL) are application workload components and belong to no IaC D-box (marked `-` in `architecture.md` section 14).

---

## Event-Driven Inference Functions Architecture

- Source diagram: [`inference-functions.drawio`](inference-functions.drawio)
- Vector export: [`inference-functions.svg`](inference-functions.svg)
- Bitmap preview: [`inference-functions.png`](inference-functions.png)

### Architectural Overview

This architecture diagram models the end-to-end event-driven inference flow across the four workload triggers: outbreak confirmation, fumigation verification, harvest estimation (triggered by contracted drone missions via DynamoDB Streams, EventBridge Pipes, a custom `drone-service` event bus, and mission-type routing rules), and telemetry batch evaluation (ingested by AWS IoT Core via parallel topic rule actions to the DynamoDB telemetry table and SQS threshold queue). Each trigger buffers into a dedicated Amazon SQS queue with adaptive visibility backoff and 14-day dead-letter queues. Four serverless AWS Lambda functions execute outside any VPC under strict domain boundaries, invoking the managed Amazon Bedrock Runtime in `us-east-1` on-demand using the Amazon Nova Pro vision-capable foundation model (zero persistent compute, zero training, and zero cross-Region inference profiles). Successful evaluations atomically persist a standardized result record under 5-year retention in DynamoDB transactional tables with conditional state transitions, while failed invocations preserve source objects permanently and leave case states unchanged. Operational visibility is centralized at an error observation point comprising dead-letter queue depths and unprocessed mission tracking across eight CloudWatch alarms, with multi-layer privacy controls guaranteeing that objects containing people are screened, excluded from inference, and routed to an isolated S3 restricted prefix under a dedicated KMS key.

### Source Documents

| Document | Sections | Topics Covered |
|---|---|---|
| `docs/inference-functions.md` | Sections 1–11 | Four triggers, Amazon Bedrock Nova Pro on-demand runtime, function contracts and strict boundaries, standardized result record schema, failure handling and state invariance, error observation point with 8 CloudWatch alarms, `us-east-1` data sovereignty, harvest volume formulas, person privacy controls, and B10 pattern |
| `docs/architecture.md` | Sections 7, 11, 14 | Workload definitions (drone service combined mode, Bedrock foundation model), pest-to-fumigation event sequence, non-VPC function placement, and D01–D13 box catalog |
| `docs/drone-service.md` | Sections 4, 5, 7, 8 | Drone mission records, reception function, verification and estimation flows, and S3 bucket prefix structure |
| `docs/farm-monitoring.md` | Sections 3, 5, 6 | Telemetry load (108,000 readings/harvest month), threshold rules evaluation, episode deduplication, and SNS alert topic |
| `docs/security-and-identity.md` | Sections 4.2, 4.3, 7.2 | Least-privilege Lambda execution roles, separation of duties (agronomist MFA authorization), and KMS customer-managed key hierarchy (`records`, `telemetry`, `objects`, `logs`, `restricted`) |
| `docs/decision-log.md` | ASM-160..169, PD-013..015, PD-111, PD-112, PD-135, PD-161 | Bedrock in `us-east-1` (ASM-160, PD-013), SQS backoff parameters (ASM-162), 0.70 confidence threshold (ASM-163), ground-station on-device person screen (ASM-164, PD-015), S3 restricted prefix and key (ASM-165), estimation formula (ASM-166), telemetry plausibility check on openings only (ASM-167), image caps (ASM-168), and pricing parameters (ASM-169) |

### Boxes D01-D13 Shown

| Box | Roadmap Issue | Component | Scope in Diagram |
|---|---|---|---|
| **D10** | #30 (B10) | Event-driven inference function pattern | Coded pattern of Epic 6: SQS threshold queue, Lambda rules engine, Alert Store, SNS Alert Topic, Bedrock plausibility check, and dead-letter queues. |
| **D09** | #29 (B09) | Telemetry ingestion pattern module | AWS IoT Core regional ingestion endpoint, dual parallel topic rule actions, DynamoDB telemetry table, and IoT rule error queue. |
| **D06** | #26 (B06) | Identities, KMS customer keys, and initial access controls | Least-privilege IAM Lambda execution roles and KMS customer-managed key hierarchy (`records`, `telemetry`, `objects`, `logs`, `restricted`). |
