# Pipeline Diagrams

This directory contains the architectural and flow diagrams for the Aroma Pacífico continuous integration, continuous delivery, telemetry ingestion, and flight-object reception pipelines.

---

## Platform CI/CD Pipeline and Release Model

- Source diagram: [`platform-pipeline.drawio`](platform-pipeline.drawio)
- Vector publication export: [`platform-pipeline.svg`](platform-pipeline.svg)
- Bitmap preview export: [`platform-pipeline.png`](platform-pipeline.png)

### Architectural Overview

The platform continuous integration and continuous delivery pipeline establishes an automated, plan-only promotion model across three isolated environments (`dev`, `qa`, `pdn`). Infrastructure and application changes originate across three decoupled repositories (`aroma-pacifico-terraform-modules`, `aroma-pacifico-infrastructure`, and `aroma-pacifico-platform`). Every merge to the `main` branch pins a single immutable Release Candidate defined by its Git SHA-256 digest in `releases.tfvars` (ASM-174). GitHub Actions jobs authenticate to AWS via OpenID Connect (OIDC) assuming dedicated, least-privilege IAM roles (`github-actions-plan-role`, `apply-role`, and an emergency `breakglass-role`) bounded by strict permission boundary policies. Infrastructure promotion enforces automated verification gates: `dev` applies automatically for continuous integration; `qa` requires passing dev tests and one peer reviewer approval; and `pdn` requires Change Advisory Board (CAB) sign-off from the SR1 Human Lead, a 24-hour advance notice to the Colombian Coffee Growers Federation, strict adherence to the harvest blackout window (05:00 to 18:00 Colombia Time in Cauca, Huila, and Tolima per ASM-183), and zero-drift plan review. Function deployments execute canary traffic shifting via AWS CodeDeploy (`LambdaCanary10Percent5Minutes`) backed by automated CloudWatch rollback alarms and forward-correction operational policies.

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `docs/automation-and-pipelines.md` | Sections 3.1–3.3, 4.1–4.4, 5, 8, 10–12 | Repositories, OIDC IAM identities, release candidate model, plan-only gates, CodeDeploy canary shifts, and rollback hooks. |
| `docs/release-notes/README.md` | All sections | Application vs. infrastructure release templates (ASM-181), harvest blackout windows, and `INC-YYYYMMDD-NN` emergency exceptions (ASM-183). |
| `docs/architecture.md` | Sections 6, 10, 14 | Automation architecture, component inventory, and authoritative D01–D13 box catalog. |
| `docs/decision-log.md` | ASM-068, ASM-171–ASM-174, ASM-181–ASM-183 | OIDC authentication, single release candidate pinning, 4-gate production promotion, and harvest window constraints. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D01` | GitHub Repositories & Workflows | Source code, reusable modules, and CI validation | Three-repo topology (`modules`, `infrastructure`, `platform`) with OIDC short-lived credentials. |
| `D02` | Remote State Backend | Isolated Terraform state persistence | Dedicated S3 bucket per environment with DynamoDB state locks and KMS CMK encryption. |
| `D03` | Secrets & Key Management | Cryptographic keys and runtime credentials | AWS Secrets Manager and KMS Customer Managed Keys with annual rotation; zero plaintext in git. |
| `D11` | CodeDeploy Application | Progressive traffic shifting engine | AWS CodeDeploy deployment group configuring `LambdaCanary10Percent5Minutes`. |
| `D12` | Deployment Pipeline Stages | Multi-environment promotion and gates | Sequential promotion (`dev` -> `qa` -> `pdn`) enforcing peer reviews and harvest window restrictions. |
| `D13` | Observability & Rollback Hooks | Automated fail-safe deployment rollback | CloudWatch error rate and latency alarms triggering automatic CodeDeploy rollback. |

---

## Telemetry Ingestion and Flight-Object Reception Pipelines

- Source diagram: [`ingestion-and-flight-objects.drawio`](ingestion-and-flight-objects.drawio)
- Vector publication export: [`ingestion-and-flight-objects.svg`](ingestion-and-flight-objects.svg)
- Bitmap preview export: [`ingestion-and-flight-objects.png`](ingestion-and-flight-objects.png)

### Architectural Overview

The message ingestion and object processing flows model two distinct left-to-right pipelines designed for high resilience, auditability, and data integrity. The Telemetry Ingestion Pipeline processes continuous agroclimatic readings (soil moisture, temperature, humidity) captured by field sensor nodes across Cauca, Huila, and Tolima. Field gateways publish telemetry over mutual TLS (mTLS) with X.509 certificates to AWS IoT Core, where an IoT Topic Rule (D09) executes direct PutItem actions into Amazon DynamoDB, bypassing intermediary compute to eliminate latency and operational overhead. DynamoDB Streams capture changes and trigger the Ingestion Processor Lambda (D06), which evaluates agronomic thresholds, deduplicates events, and dispatches critical alerts through Amazon SNS to mobile devices while buffering analytics in SQS. The Flight-Object Reception Pipeline handles high-volume multispectral drone imagery uploaded by certified operators. Ground stations authenticate via Amazon Cognito and stream encrypted mission packages via HTTPS pre-signed URLs to an S3 drone object bucket (D04) with SHA-256 integrity verification. S3 event notifications trigger the Reception Lambda (D06) for TIFF header screening and metadata indexing in DynamoDB before routing validated missions through Amazon EventBridge to Amazon Bedrock (D10). Bedrock evaluates multispectral canopy reflectance to generate coffee leaf rust (*Hemileia vastatrix*) severity heatmaps, delivering recommendations to certified agronomists for final human-in-the-loop authorization. Dedicated Dead-Letter Queues (DLQ), CloudWatch alarms, and S3 quarantine prefixes ensure corrupt or unparseable payloads never cause silent data drops.

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `docs/automation-and-pipelines.md` | Sections 6, 7, 10, 12 | Ingestion pipeline specifications, drone object reception flow, dead-letter routing, and observability hooks. |
| `docs/architecture.md` | Sections 4, 7, 10, 14 | IoT Core topic rules, DynamoDB telemetry table, S3 drone object storage, and Bedrock AI integration. |
| `docs/case/case-statement.pdf` | Challenge Point 2 & 4 | Agronomic sensor specifications, drone multispectral imaging, and coffee rust detection requirements. |
| `docs/decision-log.md` | ASM-067, ASM-073, ASM-093, ASM-138, ASM-140 | Direct IoT DynamoDB action, pre-signed upload URLs, reserved ingress tier, and threshold rules. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D04` | Drone Object Storage | Ingress and persistent storage for drone imagery | S3 bucket with KMS CMK encryption, SHA-256 validation, and lifecycle transition to Glacier. |
| `D06` | Ingestion & Reception Compute | Stream processing and metadata screening | Serverless Lambda functions with CodeDeploy canary rollouts and dead-letter error queues. |
| `D09` | IoT Core Topic Rule & Telemetry Table | High-throughput sensor telemetry ingress | AWS IoT Core MQTT broker with rule action writing directly to KMS-encrypted DynamoDB table. |
| `D10` | Bedrock Foundation Model | Plant pathology and canopy stress AI analysis | Amazon Bedrock vision foundation models executing coffee rust classification and GeoJSON scoring. |
