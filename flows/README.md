# Flows Diagrams

This directory contains architectural and operational sequence diagrams for event-driven workflows, data flows, and automated alert pipelines in the Aroma Pacífico architecture.

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
