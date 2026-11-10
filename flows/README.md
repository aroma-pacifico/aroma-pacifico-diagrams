# Flows Diagrams

This folder contains architectural workflow and sequence diagrams for event-driven processing, data flows, and automated pipelines in the Aroma Pacífico pilot architecture.

---

## Event-Driven Inference Functions Architecture

- **Primary Source:** [`inference-functions.drawio`](inference-functions.drawio)
- **Vector Publication Export:** [`inference-functions.svg`](inference-functions.svg)
- **High-Resolution Preview:** [`inference-functions.png`](inference-functions.png)

### What the diagram shows

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

### Diagram Boxes D01-D13 Shown

| Box | Roadmap Issue | Component | Scope in Diagram |
|---|---|---|---|
| **D10** | #30 (B10) | Event-driven inference function pattern | Coded pattern of Epic 6: SQS threshold queue, Lambda rules engine, Alert Store, SNS Alert Topic, Bedrock plausibility check, and dead-letter queues. |
| **D09** | #29 (B09) | Telemetry ingestion pattern module | AWS IoT Core regional ingestion endpoint, dual parallel topic rule actions, DynamoDB telemetry table, and IoT rule error queue. |
| **D06** | #26 (B06) | Identities, KMS customer keys, and initial access controls | Least-privilege IAM Lambda execution roles and KMS customer-managed key hierarchy (`records`, `telemetry`, `objects`, `logs`, `restricted`). |
