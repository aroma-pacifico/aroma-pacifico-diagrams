# Flows Diagrams

This folder contains architectural workflow and sequence diagrams for event-driven processing, data flows, and automated pipelines in the Aroma Pacífico pilot architecture.

---

## Event-Driven Inference Functions Architecture

- Source diagram: [`inference-functions.drawio`](inference-functions.drawio)
- Vector export: [`inference-functions.svg`](inference-functions.svg)
- Bitmap preview: [`inference-functions.png`](inference-functions.png)

### What the diagram shows

1. **The Four Event-Driven Triggers (Section 3)**:
   - **Outbreak Confirmation Object (`Objeto de confirmación del foco`)**: Triggered when the reception function (A15) sets `record_persisted` on the DynamoDB `missions` table for a confirmation mission. A DynamoDB Stream feeds an Amazon EventBridge Pipe that filters `record_persisted` events and forwards them to the custom EventBridge bus `drone-service`. An EventBridge rule on `mission_type = confirmation` targets the SQS Confirmation Queue (`<prefix>-pdn-sqs-infconfirm`).
   - **Verification Object (`Objeto de verificación`)**: Triggered by `mission_record_persisted` for a verification mission through the same Pipe and EventBus, filtered by an EventBridge rule on `mission_type = verification` targeting the SQS Verification Queue (`<prefix>-pdn-sqs-infverify`).
   - **Estimation Flight Object (`Objeto del vuelo de estimación`)**: Triggered by `mission_record_persisted` for an estimation mission through the same Pipe and EventBus, filtered by an EventBridge rule on `mission_type = estimation` targeting the SQS Estimation Queue (`<prefix>-pdn-sqs-infestimate`).
   - **Telemetry Batch (`Lote de telemetría`) [B09 / D09 Pattern]**: Accepted by AWS IoT Core on topic `aroma/telemetry/v1/...`. An IoT topic rule executes two parallel decoupled actions: Action 1 writes the reading to DynamoDB `telemetry` table (under KMS key `telemetry`), while Action 2 forwards the reading key and payload to the SQS Threshold Queue (`<prefix>-pdn-sqs-infthreshold`). If either action fails, payload is routed to `<prefix>-pdn-sqs-ioterror`.

2. **Managed Inference Service in `us-east-1` (Section 2)**:
   - Service: Amazon Bedrock Runtime invoked on-demand in `us-east-1` (N. Virginia, USA).
   - Initial Model: Amazon Nova Pro (`amazon.nova-pro-v1:0`), offering in-Region vision-capable foundation model capabilities (ASM-160).
   - Invocation Mode: Per-request invocation via Bedrock `Converse` API using versioned zero-shot system prompts and structured JSON output schemas.
   - Zero Persistent Compute: Meets Part E requirements ("No se entrena un modelo" and "No hay cómputo persistente en espera de objetos"). Zero training, zero fine-tuning, zero provisioned throughput, and no SageMaker real-time endpoints.
   - Regional & Routing Safeguards: Lambda execution roles hold `bedrock:InvokeModel` strictly on `arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-pro-v1:0` with explicit deny on all cross-Region inference profile ARNs (`arn:aws:bedrock:*::inference-profile/*`). Root SCP denies non-us-east-1 regions.

3. **The Four Lambda Functions & Boundaries (Sections 4 & 11)**:
   - Common 7-step function contract: SQS event receive (identifiers only, zero PII) -> idempotency check (query prior result; delete message and exit if found) -> guard check (case/order/mission/node state; record `guard_skipped` if false) -> read persisted input (verify `person_screen = clear`) -> invoke Bedrock -> conditional DynamoDB persistence -> delete SQS message and emit metrics.
   - **Confirmation Function (`inf-confirmation`)**: Reads up to 8 focus photos by version/SHA-256 with `person_screen = clear`. Evaluates `pest_signs_present` against 0.70 confidence threshold (ASM-163). Writes `confirmation_result` and conditionally transitions case `confirming` to `confirmed` or `not_confirmed`. CRITICAL BOUNDARY: NEVER authorizes or creates a fumigation order (`authorizer_id` is human agronomist with MFA only).
   - **Verification Function (`inf-verification`)**: Reads up to 8 short-pass frames (`person_screen = clear`) and track geometry (verifies pass covered focus; track not sent to Bedrock). Evaluates `active_signs_at_focus`. Conditionally transitions case `verifying` to `closed` / `verification_not_closable` and order `executed` to `closed`. CRITICAL BOUNDARY: NEVER closes a case without a persisted verification object.
   - **Estimation Function (`inf-estimation`)**: Reads up to 20 plot photos and 1 plot map (`person_screen = clear`) with farm active polygon. Evaluates plot productive cover fraction and crop load class, computing `estimated_sacks = area_ha x cover x yield_factor`. Writes harvest estimate record and updates Collection Center expected-sacks view. CRITICAL BOUNDARY: NEVER creates packing orders, sacks, or code batches.
   - **Telemetry Threshold Function (`inf-threshold`) [B10 / D10 Pattern]**: Reads stored reading via strongly consistent `GetItem` (retried 3 times if missing: 200, 400, 800 ms). Evaluates dynamic Federation threshold version in force at original reading timestamp. Writes evaluation item to Alert Store and publishes immediately to Amazon SNS Alert Topic (< 5-minute latency budget). Bedrock Nova Pro plausibility check is ONLY invoked on episode openings AFTER SNS publication. CRITICAL BOUNDARY: Bedrock NEVER suppresses, delays, or reclassifies an alert, and NEVER changes a pest case. Role holds NO write permission on telemetry table.

4. **Result Record ("Expediente") Schema (Section 5)**:
   - Persisted in DynamoDB transactional stores (`cases`, `estimates`, and `alerts` tables) encrypted with KMS key `records`.
   - Standard fields: `result_id` (idempotency key), `function`, `outcome`, `result_at` (UTC timestamp), `confidence`, `confidence_basis` (`model` / `rule`), `source` (bucket, key, version, SHA-256 or table, node, timestamp), `excluded_objects`, `model_id`, `model_region` (`us-east-1`), `prompt_version`, `bedrock_request_id`, `input_tokens`, `output_tokens`, `event_id`.
   - Retention: 5 years Object Lock compliance (ASM-098, ASM-154).

5. **Failure Handling, Retries & Dead-Letter Paths (Section 6)**:
   - Multi-layer retry: AWS SDK adaptive retry (3 attempts) on Bedrock throttling/errors; SQS visibility backoff (60, 120, 240, 480 s for drones; 10, 20, 40, 80 s for telemetry); `maxReceiveCount = 5` before sending to DLQ.
   - Dedicated Dead-Letter Queues (DLQs): `<prefix>-pdn-sqs-infconfirmdlq`, `infverifydlq`, `infestimatedlq`, `infthresholddlq` (14-day retention, KMS `records`), plus EventBridge Pipe DLQ, Rule DLQs, and IoT error queue.
   - Manual SQS DLQ redrive by platform operator after fix; idempotency guarantees safe replay without re-invoking Bedrock or duplicate state changes.
   - Source Object Permanence: Functions hold NO `s3:PutObject` / `s3:DeleteObject` and NO write permission on `telemetry` table. On failure, source objects remain permanent and case state remains unchanged ("el objeto de origen permanece y el estado del caso no cambia").

6. **Error Observation Point & CloudWatch Alarms (Section 7, Point 6)**:
   - Proposed CloudWatch Alarms for A21 (PC-IAC-003 naming):
     - DLQ depth > 0: `<prefix>-pdn-alarm-infconfirmdlq`, `infverifydlq`, `infestimatedlq`, `infthresholddlq`.
     - Upstream route failures: `<prefix>-pdn-alarm-infroutedlq`.
     - Queue latency: `<prefix>-pdn-alarm-infconfirmage`, `infverifyage`, `infestimateage`, `infthresholdage`.
     - Invocations failed without state change: `<prefix>-pdn-alarm-inffailed` (metric: `InferenceFailed` in `AromaPacifico/Inference`).
     - Missions awaiting result: `<prefix>-pdn-alarm-infawaiting` (metric: `MissionsAwaitingResult`).
     - Store write outage: `<prefix>-pdn-alarm-infthresholdstore` (metric: `ReadingNotPersisted` and IoT error queue depth).
     - Model throttling: `<prefix>-pdn-alarm-infthrottle` (metric: `InferenceThrottled` and `AWS/Bedrock` throttles).
     - Person detected in crop object: `<prefix>-pdn-alarm-infperson` (CRITICAL incident: `ModelPersonDetected > 0`).
   - Structured JSON logging under KMS key `logs` (zero PII, zero images, audit request IDs).

7. **Treatment of Objects Containing a Person (Section 10 | PD-015, ASM-164, ASM-165)**:
   - On-device person detector on ground-station equipment screens all photos before upload; manifest marks `person_screen = clear` or `person_present`.
   - Platform fail-closed: missing, unknown, or non-clear entries treated as `not_screened` and strictly excluded from inference.
   - Reception function moves `person_present` objects to `restricted/person/{yyyy}/{mm}/{dd}/{key}` under dedicated KMS key `restricted` (8th key class). Explicit deny on `restricted/*` for all Lambda functions and human roles.
   - Cloud backstop: If Bedrock returns `person_detected = true`, result is discarded (`outcome = not_evaluable`), object is moved to `restricted/`, and `ModelPersonDetected` incident alarm fires.

8. **Data Sovereignty & What Leaves Colombia (Section 8.2 | PD-013, ASM-002)**:
   - Stored and processed in `us-east-1` (USA) because no AWS Region exists in Colombia.
   - Clear photos and opening telemetry readings sent to Bedrock in `us-east-1`. Personal data, tracks, delimitation objects, and routine telemetry readings are NEVER sent to Bedrock.

---

### Source Documents

- `aroma-pacifico-architecture/docs/inference-functions.md`: all sections:
  - Section 1 (Purpose and requirements)
  - Section 2 (Managed inference service, Amazon Bedrock, Nova Pro, prompt schemas)
  - Section 3 (The four triggers: confirmation, verification, estimation, telemetry batch)
  - Section 4 (Function design: common contract, confirmation, verification, estimation, telemetry threshold)
  - Section 5 (Result record: schema, idempotency, Object Lock 5-year retention)
  - Section 6 (Failure handling: SDK retry, SQS backoff, DLQs, state invariance, idempotency, telemetry ordering)
  - Section 7 (Error observation point: 8 proposed CloudWatch alarms, custom metrics, structured logging)
  - Section 8 (Region `us-east-1`, foundation model restriction, data sovereignty matrix)
  - Section 9 (Harvest volume: 108,000 readings, 200 estimation flights, pricing formula, cost drivers)
  - Section 10 (Treatment of objects with a person: ground-station screen, fail-closed, S3 restricted prefix, KMS `restricted`)
  - Section 11 (Epic 6 function pattern: telemetry threshold coded by B10, drone functions architecture-only)
  - Section 12 (Compliance with acceptance criteria)
  - Section 13 (Decision-log entries closed: PD-013, PD-014, PD-015, PD-111, PD-112, PD-135; ASM-160..ASM-169)
  - Section 14 (References)
- `aroma-pacifico-architecture/docs/architecture.md`:
  - Section 7 (Point 5 Workloads: drone service combined mode, Bedrock foundation model)
  - Section 11 (Pest-to-fumigation flow)
  - Section 14 (Diagram boxes D01-D13 catalog and traceability to IaC)
- `aroma-pacifico-architecture/docs/drone-service.md`: Sections 4, 5, 7, 8 (records, states, S3 prefixes, triggers)
- `aroma-pacifico-architecture/docs/farm-monitoring.md`: Sections 3, 5, 6 (load, threshold evaluation, episode deduplication, SNS alert topic)
- `aroma-pacifico-architecture/docs/security-and-identity.md`: Sections 4.2, 4.3, 7.2, 9 (role matrix, separation of duties, KMS customer keys, personal data isolation)
- `aroma-pacifico-architecture/docs/decision-log.md`:
  - `ASM-002` (Platform in us-east-1)
  - `ASM-097`, `ASM-098`, `ASM-154` (KMS customer-managed keys, 5-year retention)
  - `ASM-140` (Drone ground station at Support VLAN 102, 10.4x.2.10)
  - `ASM-149` (Reception function writes record_persisted)
  - `ASM-160` (Amazon Bedrock Nova Pro in us-east-1, zero-shot prompts, no custom training)
  - `ASM-161` (Delivery paths: DynamoDB stream, Pipe, EventBus, Rule, SQS; IoT rule actions)
  - `ASM-162` (Failure parameters: timeouts, memory, concurrency, visibility, backoff)
  - `ASM-163` (0.70 confidence threshold)
  - `ASM-164` (On-device person screen at ground station before upload; fail-closed)
  - `ASM-165` (S3 restricted prefix and dedicated KMS key `restricted`)
  - `ASM-166` (Estimation formula and Federation yield factors)
  - `ASM-167` (Telemetry plausibility check on episode openings only, after SNS publish)
  - `ASM-168` (Image caps per invocation: <= 8 confirmation/verification, <= 21 estimation)
  - `ASM-169` (Named pricing parameters for A23 cost model)
  - `ADR-001` (Primary cloud AWS us-east-1, Azure prepared)
  - `ADR-002` (Drone combined mode, agronomist MFA authorization)
  - `ADR-003` (IaC pattern module, IoT Core ingestion, DynamoDB telemetry table, SNS alert topic)
  - `PD-013`, `PD-014`, `PD-015`, `PD-111`, `PD-112`, `PD-135`, `PD-161` (Closed decisions)
  - `PD-018` (Open: AWS live feature re-check)

---

### Diagram Boxes D01-D13 Shown

- **`D10`** (Track B issue #30 / B10): The Event-Driven Inference Function Pattern (SQS threshold queue, Lambda rules engine, Alert Store, SNS Alert Topic, Bedrock plausibility check, DLQs).
- **`D09`** (Track B issue #29 / B09): Telemetry Ingestion Pattern Module (AWS IoT Core, topic rules, DynamoDB telemetry table, IoT rule error SQS queue).
- **`D06`** (Track B issue #26 / B06): Identity & Access Controls (IAM Lambda execution roles with least-privilege, KMS customer-managed keys `records`, `telemetry`, `objects`, `logs`, `restricted`).
- **`D04`** (Track B issue #24 / B04): VPC & Application Tier (private subnets across 3 AZs, VPC interface endpoints for SQS, Bedrock, S3, DynamoDB, KMS, CloudWatch Logs).
