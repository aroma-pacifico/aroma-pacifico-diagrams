# Flows Diagrams

This folder contains architectural and operational sequence diagrams for event-driven workflows, data flows, and automated alert pipelines in the Aroma Pacífico architecture.

---

## Farm Monitoring and Pilot Alerts Sequence

- Source diagram: [`farm-monitoring-sequence.drawio`](farm-monitoring-sequence.drawio)
- Vector export: [`farm-monitoring-sequence.svg`](farm-monitoring-sequence.svg)
- Bitmap preview: [`farm-monitoring-sequence.png`](farm-monitoring-sequence.png)

### What it shows

1. **End-to-End Event Sequence**:
   - Traces the end-to-end flow from initial environmental measurement at the field node through ingestion, storage, rules evaluation, episode deduplication, and notification dispatch to farmer receipt.
   - Covers both crop monitoring (6 readings/day per farm across 200 pilot farms) and harvest drying monitoring (24 readings/day while drying lot is active across 100 farms), scaling to 1,000 farms without architectural redesign.

2. **The 5-Minute Latency Budget Allocation**:
   - Demonstrates the rigorous 7-hop latency budget defined in `docs/connectivity.md` section 8.2 and `docs/farm-monitoring.md` section 7.2:
     - **Hop 1 (10 s)**: Sensor reading taken to message ready on device (nodes publish immediately, holding zero threshold logic).
     - **Hop 2 (60 s)**: Radio wake, network attach (LTE-M, NB-IoT, Cat-1, or satellite IP bearer), and mutual TLS 1.2+ handshake using unique X.509 client certificate to regional endpoint in `us-east-1` (port 8883 / 443 ALPN).
     - **Hop 3 (20 s)**: MQTT Publish QoS 1 to `PUBACK` at AWS IoT Core endpoint; local encrypted buffer is released only upon `PUBACK`.
     - **Hop 4 (10 s)**: AWS IoT Core topic rule execution executing dual parallel actions: direct write to DynamoDB `telemetry` table (A08 audit requirement) and enqueueing message key to SQS threshold queue (with DLQ).
     - **Hop 5 (20 s)**: Threshold Lambda execution: strongly consistent read from telemetry table (ASM-161), querying active Federation threshold version from DynamoDB `thresholds` table at `original_timestamp` (zero cache), episode deduplication in DynamoDB Alert Store, and publishing alert event to Amazon SNS Alert Topic.
     - **Hop 6 (30 s)**: SNS Alert Topic fanout to SQS notification queue; Notification Dispatcher Lambda resolves farmer phone and push endpoint from DynamoDB `personal-data` table (KMS `personal`, strict PII isolation) and applies SMS coalescing.
     - **Hop 7 (60 s)**: Parallel multi-channel dispatch via AWS End User Messaging SMS and Amazon SNS Mobile Push (FCM / APNs) to farmer phone.
     - **Totals**: Target Sum = 210 s (3.5 min), Retry & Network Jitter Margin = 90 s (1.5 min), Case Statement SLA Limit = 300 s (< 5 minutes).

3. **Store-and-Forward and Offline Drain**:
   - Outlines the no-signal behavior: readings buffered in non-volatile encrypted ring buffer (max 24 h). Evicted unacknowledged readings (> 24 h) are counted and reported via `telemetry_gap` records.
   - Reconnection drain: live reading sent first, followed by buffered readings drained oldest first. Drained records evaluated at their `original_timestamp`.
   - Delayed flag: Readings with `arrival_time - original_timestamp > 300 s` are flagged `delayed = true`, enabling the farmer to distinguish live from buffered alerts.

4. **Federation Thresholds as Configurable Data**:
   - Captures the initial ranges from the case statement table (T-01 to T-10) for pH, air/soil moisture, air/soil temperature, NPK, drying coffee moisture, and drying grain temperature.
   - Thresholds stored in DynamoDB `thresholds` table (KMS `records`) with composite key `rule_key` (PK) and `effective_from` (SK).
   - Scope resolution: farm-specific override takes precedence over pilot rule; zero caching ensures immediate applicability to subsequent readings.
   - Complete audit trail: application version item authoritative, DynamoDB Streams delivery to S3 Object Lock archive, and CloudTrail data events (write-only).

5. **Episode Model & Deduplication**:
   - Solves alert fatigue across 108,000 readings/harvest month by making the episode the unit of alerting, not the raw reading.
   - First out-of-range reading opens an Episode item using a conditional put (`attribute_not_exists`) and emits `crop_alert_raised` (triggering ADR-002 pest case mode switch for drone confirmation).
   - Subsequent out-of-range readings attach to open episode (updating count and extremes, suppressing redundant notifications).
   - In-range reading closes episode (`alert_resolved`) via push notification.
   - 24-hour reminder sent by push only if episode remains open.
   - Drying coffee moisture <= 12% emits single `drying_closure_notice` per lot.

6. **Notification Dispatch, PII Isolation, and Farmer Perception**:
   - Channel decision (closes PD-085, ASM-136): dual channel (Push + SMS) for alert openings and closure notices; push only for reminders and resolutions.
   - SMS reaches farmer over cellular network without requiring data balance; Push provides rich history and curves when phone enters data coverage.
   - SMS coalescing: first SMS sent immediately; subsequent alerts for same farm within 10 minutes combined into one follow-up SMS.
   - Strict PII isolation: Threshold function never sees farmer names, phones, or credentials; Dispatcher reads DynamoDB `personal-data` under KMS `personal` and never sees raw telemetry.
   - What the farmer sees (sec 8): plain text SMS with farm alias only (no PII, no URLs), and 3 Farmer App views (Farm Status Card, Alerts List with measured/received timestamps, History Charts with recommended bands).
   - Delivery receipts: carrier SMS receipt and app push receipt update episode in Alert Store, providing the SLI metric for the Point 6 alert-delivery SLO (A21, PD-134).

---

### Source Documents

- `aroma-pacifico-architecture/docs/farm-monitoring.md`:
  - Section 1 (Purpose and requirements)
  - Section 3 (Load: harvest day 3,600 readings, 5-year volume, 1,000 farms scale)
  - Section 4 (Thresholds as Federation data, initial ranges T-01..T-10, configuration model, change audit)
  - Section 5 (Evaluation of a reading, step-by-step classification, persistence)
  - Section 6 (Alerts and deduplication, episode model, buffer drain, drying lot)
  - Section 7 (Farmer notification channel, push and SMS decision, path, latency within budget)
  - Section 8 (What the farmer sees, 3 app screens, sample SMS alerts)
  - Section 9 (History and the pilot report)
  - Section 10 (Observability hooks and delivery receipts)
- `aroma-pacifico-architecture/docs/connectivity.md`:
  - Section 4.2 & 4.3 (Exit protocol, mutual TLS, 24-hour non-volatile buffer, store-and-forward drain)
  - Section 4.4 (Out-of-coverage farms and satellite IP bearer)
  - Section 8 (Crop alert under 5 minutes, 7-hop latency budget, tie to alert-delivery SLO)
  - Section 11 (Mobile network generations: LTE-M / NB-IoT / Cat-1 or later, zero 2G/3G dependency)
- `aroma-pacifico-architecture/docs/architecture.md`:
  - Section 11 (Pest-to-fumigation flow, `crop_alert_raised` trigger)
  - Section 12 (Store-and-forward and no-signal behavior matrix)
  - Section 14 (Diagram boxes D01-D13 catalog and traceability to IaC)
- `aroma-pacifico-architecture/docs/decision-log.md`:
  - `ASM-053`, `ASM-054`, `ASM-059` (Soil pH, NPK publication status, drying closure notice)
  - `ASM-067` (5-year audit retention under Object Lock)
  - `ASM-070` (Two nodes per farm with dedicated identities)
  - `ASM-074` (24-hour buffer, live reading first, gap report)
  - `ASM-075` (Per-hop latency budget targets)
  - `ASM-091`, `ASM-092` (Passwordless SMS for field users, fresh auth for operator)
  - `ASM-097` (KMS customer-managed key per data class: `telemetry`, `records`, `personal`)
  - `ASM-130` to `ASM-138` (Drying lot control, separate air/soil ranges, closure tolerance, recommended bands, episode deduplication, dual-channel notification, 5-year threshold retention, threshold scopes and timestamp evaluation)
  - `ASM-161` (SQS threshold queue decoupling and strongly consistent read)
  - `ASM-162` (Lambda failure and concurrency parameters)
  - `ASM-167` (Bedrock Nova Pro plausibility check on opening readings)
  - `ADR-002` (Drone combined mode and `crop_alert_raised` pest case opening)
  - `ADR-003` (IaC pattern module, IoT Core ingestion, DynamoDB telemetry table, SNS alert topic)
  - `PD-074` (Closed: DynamoDB threshold store and audit)
  - `PD-085` (Closed: Push and SMS dual-channel alert decision)
  - `PD-130`, `PD-134`, `PD-135` (Quota verification, delivery receipts SLI, threshold evaluation contract)

---

### Diagram Boxes D01-D13 Shown

- **`D08`** (Track B issue #28 / B08): Reusable Farm Module (field node, crop and drying sensors, 24h buffer, device X.509 certificate).
- **`D09`** (Track B issue #29 / B09): Telemetry Ingestion Pattern Module (AWS IoT Core, topic rules, DynamoDB telemetry table, error SQS queue).
- **`D10`** (Track B issue #30 / B10): Event-Driven Inference & Function Pattern (SQS threshold queue, Lambda rules engine, DynamoDB Alert Store, SNS Alert Topic).
- **`D06`** (Track B issue #26 / B06): Identity & Access Controls (device X.509 certificates, IAM execution roles, KMS customer-managed encryption keys `telemetry`, `records`, `personal`).
