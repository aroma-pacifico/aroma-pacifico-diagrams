# Pipeline Diagrams

This directory contains the architectural and flow diagrams for the Aroma Pacífico continuous integration, continuous delivery, telemetry ingestion, and flight-object reception pipelines. All diagrams are authored as Draw.io XML sources and exported to vector SVG and high-resolution PNG formats.

---

## Platform CI/CD Pipeline and Release Model

- Source diagram: [`platform-pipeline.drawio`](platform-pipeline.drawio)
- Vector publication export: [`platform-pipeline.svg`](platform-pipeline.svg)
- Bitmap preview export: [`platform-pipeline.png`](platform-pipeline.png)

### Architectural Overview

The platform continuous integration and continuous delivery pipeline establishes an automated, plan-only promotion model across three isolated environments (`dev`, `qa`, and `pdn`). Infrastructure and application changes originate across three decoupled repositories: the public reference modules (`aroma-pacifico-terraform-modules`), the private environment root stacks (`aroma-pacifico-infrastructure`), and the application workload codebase (`aroma-pacifico-platform`, proposed under ASM-170 / PD-170). Every release tag builds an immutable Release Candidate whose byte-identical package digests are pinned in `stacks/workload/releases.tfvars` (ASM-171). GitHub Actions workflows authenticate to AWS using OpenID Connect (OIDC) assuming dedicated, least-privilege IAM roles (`plan` role for read-only PR checks, `apply` role for automated deployment on merge). In accordance with the project security model, the emergency `BreakGlass` permission set operates solely through AWS IAM Identity Center and is never an automated deployment path.

Promotion follows a strictly sequential three-stage pipeline. The development environment (`dev`) executes continuous integration via automated plan and apply upon merge to `main`. The quality assurance environment (`qa`) serves as the pattern environment (D04, D05, D08), requiring dev test logs, one peer review among SR1, SR2, JR1, or JR2 (excluding the author, ASM-172), and plan diff review. Production promotion (`pdn`) requires passing four mandatory gates: authorization from one of SR1 or SR2 (excluding the author and triggering person, ASM-172); advance notice to the Colombian Coffee Growers Federation with a default lead time of 2 business days (ASM-173, Q-15); execution outside the coffee harvest window (05:00 to 18:00 Colombia time on harvest-week days, and refusing to start if expected end passes 05:00; ASM-007, ASM-178); and verification of a fresh plan against recorded QA acceptance (automation-and-pipelines.md section 4.4). Emergency changes inside the window or with shortened notice require an incident declaration by SR1 or SR2 with an incident reference (INC-YYYYMMDD-NN) recorded in the approval and release note, strictly to repair critical operational harm (ASM-183). The pipeline operationalizes automated CI plan generation and verification gates (D11, section 4.2), multi-environment promotion (D12, sections 4.4, 8, 9), and plan evidence collection (D13, PD-019). Function deployments execute canary traffic shifting via AWS CodeDeploy (`LambdaCanary10Percent5Minutes`) backed by automated CloudWatch rollback alarms and forward-correction schema policies (ASM-174, ASM-175, ASM-185).

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `docs/automation-and-pipelines.md` | Sections 3.1–3.3, 4.1–4.4, 8, 9, 10–12 | Repositories, OIDC IAM identities, release candidate model, promotion gates, approvers (ASM-172), notice lead time (ASM-173), CodeDeploy canary shifts, and rollback targets (ASM-174). |
| `docs/release-notes/README.md` | Sections 2–4, 6 | Application and infrastructure release notes, harvest blackout windows, and incident declaration rules (ASM-183, ASM-184). |
| `docs/architecture.md` | Sections 6, 10, 14 | Automation architecture, component inventory, and authoritative D01–D13 box catalog. |
| `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` | Repository layout, Remote state, CI and CD constraints | Terraform versions, S3 backend with native locking, OIDC pipeline roles, and telemetry pattern module. |
| `docs/decision-log.md` | ASM-007, ASM-097, ASM-170–ASM-175, ASM-178, ASM-183, ASM-185, ASM-229 | OIDC credentials, release pinning, approver matrix, 2-day notice, rollback targets, harvest blackout, and schema compatibility. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D01` | GitHub Repositories & Workflows | Source code, reusable modules, and CI validation | Three-repository topology (`modules`, `infrastructure`, `platform`) with GitHub Actions OIDC authentication. |
| `D02` | Remote State Backend | Isolated Terraform state persistence | Dedicated S3 bucket per environment with native state locking and KMS CMK encryption (`state` key class). |
| `D03` | Configuration & Secrets | Cryptographic keys and runtime credentials | AWS Secrets Manager and SSM Parameter Store; dynamic resolution at runtime with zero credentials in git or state. |
| `D06` | Pipeline IAM Identities | Scoped execution permissions | GitHub Actions OIDC assuming dedicated `plan` and `apply` roles; BreakGlass reserved for console emergencies. |
| `D11` | CI Pipeline & Plan Gates | Automated reviewable plan generation | GitHub Actions CI producing an executable, reviewable `terraform plan` on pull requests (section 4.2). |
| `D12` | Deployment Pipeline Stages | Multi-environment promotion and gates | Sequential promotion (`dev` -> `qa` -> `pdn`) enforcing peer reviews, Federation notice, and harvest window checks (sections 4.4, 8, 9). |
| `D13` | Plan Validation & Evidence | Validation of executable plan | Automated verification checks (`fmt`, `validate`, `tflint`, security scans) and attested plan evidence in pull requests (PD-019, section 4.2). |

---

## Telemetry Ingestion and Flight-Object Reception Pipelines

- Source diagram: [`ingestion-and-flight-objects.drawio`](ingestion-and-flight-objects.drawio)
- Vector publication export: [`ingestion-and-flight-objects.svg`](ingestion-and-flight-objects.svg)
- Bitmap preview export: [`ingestion-and-flight-objects.png`](ingestion-and-flight-objects.png)

### Architectural Overview

The message ingestion and object processing architecture models two event-driven pipelines designed for data integrity, privacy compliance, and resilience. The Telemetry Ingestion Pipeline processes continuous agroclimatic readings (soil moisture at 10cm and 30cm, soil temperature, air temperature, relative humidity, and NPK) transmitted by field sensor nodes across 200 coffee farms in Cauca, Huila, and Tolima (ASM-070). Nodes publish every reading immediately over cellular LPWA (LTE-M/NB-IoT) with MQTT over TLS 1.2 or later and mutual X.509 certificate authentication to AWS IoT Core in `us-east-1` (ADR-003, `connectivity.md`). In accordance with ADR-003 and ASM-161, the IoT Topic Rule executes two direct actions: a PutItem write into the KMS-encrypted DynamoDB `telemetry` table and an SQS message action to the threshold evaluation queue (`infthresholdqueue`). The rule error action routes unparseable or failed messages to a dedicated SQS Dead-Letter Queue with up to 5 attempts (ASM-162). The Threshold Lambda function (D10) reads from the SQS queue, fetches the reading strongly consistent from DynamoDB (ASM-161), and evaluates values against Federation-configured threshold data stored in the DynamoDB `thresholds` table (farm scope checked first, fallback to pilot scope; `farm-monitoring.md` section 4). If a threshold is exceeded, the function persists a `crop_alert_raised` record and emits an event to the SNS alert topic to notify the farmer via SMS and mobile push within 5 minutes (ASM-075).

The Flight-Object Reception Pipeline handles drone mission packages uploaded by certified commercial operators (ADR-002, `drone-service.md`). Before upload, the ground station software executes an on-device person screen on every photo, frame, and map, populating the `person_screen` manifest field (`clear` or `person_present`, ASM-164). Ground stations authenticate via Amazon Cognito (`staff` pool with MFA) and upload mission packages to the workload S3 drone bucket using time-limited Pre-signed URLs with SHA-256 header checksum verification. Files containing persons are immediately relocated by the Reception Lambda to `restricted/person/` under an eighth KMS key class `restricted` with an explicit Deny policy for all function roles (ASM-165). Validated missions are recorded in the DynamoDB `missions` table with status `record_persisted`, returning a signed receipt to the ground station to release its local buffer. The DynamoDB stream routes `record_persisted` events through an EventBridge Pipe to the drone custom event bus, which dispatches events to dedicated SQS queues per mission type (batch size 1, ASM-161). The inference functions invoke Amazon Bedrock on demand in `us-east-1` using the foundation-model identifier of a vision-capable model, initially Amazon Nova Pro (ASM-160). Bedrock evaluates canopy condition with zero-shot prompts and JSON schemas, returning an uncalibrated confidence score (ASM-163). A focus is confirmed and a case closed only if confidence is 0.70 or greater; all outputs are persisted with timestamp, confidence, and source object key without ever authorizing fumigation or closure directly (ASM-160, ASM-163). Certified agronomists review AI findings and authorize prescriptions with MFA (ADR-002).

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `docs/automation-and-pipelines.md` | Sections 6, 7, 10, 12 | Ingestion pipeline specifications, drone object reception flow, dead-letter queue routing, and rollback alarm hooks. |
| `docs/farm-monitoring.md` | Sections 3, 4.1–4.3, 5, 6.6 | Telemetry payload, DynamoDB telemetry table, versioned DynamoDB `thresholds` table, and threshold evaluation flow. |
| `docs/drone-service.md` | Sections 3, 7, 8, 9 | Mission package format, pre-signed upload grants, SHA-256 verification, 5-year S3 lifecycle retention, and quarantine rules. |
| `docs/inference-functions.md` | Sections 2, 3, 4, 6.1, 10.2, 10.3 | Amazon Nova Pro foundation model (ASM-160), EventBridge Pipe and SQS routing (ASM-161), failure design values (ASM-162), confidence thresholds (ASM-163), on-device person screen (ASM-164), and `restricted` key class (ASM-165). |
| `docs/connectivity.md` | Sections 4.1–4.3, 8.2 | Field sensor nodes, cellular LPWA radio, MQTT over TLS 1.2 or later, QoS 1 on port 8883, 24-hour non-volatile store-and-forward buffer, and 5-minute alert budget (ASM-075). |
| `docs/architecture.md` | Sections 4, 7, 10, 14 | Telemetry and drone workloads, component inventory, and authoritative D01–D13 box catalog. |
| `docs/adr/adr-002-drone-flight-mode-and-governance.md` | Flight mode, Human authorization | Combined flight mode, certified pilot operations, and mandatory agronomist MFA authorization. |
| `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` | Pattern module: telemetry ingestion | IoT Core topic rule direct PutItem action, SQS threshold action, and SNS alert topic. |
| `docs/decision-log.md` | ASM-070, ASM-075, ASM-097, ASM-160–ASM-165 | Node counts, alert latency, KMS key classes, Nova Pro model, event routing, retry limits, confidence threshold, person screen, and quarantine. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D08` | Field Sensor Node Pattern | Reusable farm module | Reusable farm telemetry collection: two nodes per farm, cellular LPWA radio, mutual TLS 1.2+ with X.509 certs, and 24-hour non-volatile buffer (ADR-003, `connectivity.md`). |
| `D09` | IoT Topic Rule & Telemetry Table | High-throughput sensor telemetry ingress | AWS IoT Core MQTT broker, SQL topic rule executing direct PutItem to KMS-encrypted DynamoDB `telemetry` table and SQS message action to threshold queue (ADR-003, ASM-161). |
| `D10` | Event-Driven Inference Functions | Agroclimatic evaluation and image inference | Amazon Bedrock vision model (Amazon Nova Pro in `us-east-1`, ASM-160) and event-driven Threshold Lambda function (ASM-161, ASM-163). |
