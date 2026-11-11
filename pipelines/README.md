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

---

## Infrastructure as Code CI and Reusable Module Release Sequence

- Source diagram: [`iac-ci-sequence.drawio`](iac-ci-sequence.drawio)
- Vector publication export: [`iac-ci-sequence.svg`](iac-ci-sequence.svg)
- Bitmap preview export: [`iac-ci-sequence.png`](iac-ci-sequence.png)

### Architectural Overview

The Infrastructure as Code Continuous Integration sequence defines the mandatory quality and security gates evaluated on every pull request targeting `main`. Operating strictly under Trunk-Based Development (AGENTS.md section 6), developers work on short-lived branches (`<type>/<issue>-<slug>`) and commit atomically using Conventional Commits with a mandatory cross-repository issue reference (`Refs aroma-pacifico/aroma-pacifico-roadmap#<n>`). Opening a pull request triggers automated GitHub Actions workflows in parallel with non-colliding concurrency cancellation.

Validation follows the principle of cheapest gate first:
1. **Pull Request Hygiene & Zero-Emoji Check**: Enforces Conventional Commit headers, English-only prose, valid branch names, and strict zero-emoji compliance (`pr-conventions.yml`).
2. **Secret Scanning (`env-secrets`)**: Scans all `*.tfvars`, `*.tfbackend`, and `*.example` files for private keys, AWS credentials, ARNs, and sensitive identifiers (`scripts/check-env-secrets.sh`, PC-IAC-016).
3. **Multi-Stack Validation Matrix (`gates`)**: Evaluates the four root stacks (`state-backend`, `network`, `security`, and `workload`) in parallel across six automated gates: syntax formatting (`terraform fmt -check -diff`), backend-free initialization (`terraform init -backend=false`), schema validation (`terraform validate`), linting against the 28 `pc-iac` rules (`tflint --init --config=.tflint.hcl`), security misconfiguration scanning (`aquasecurity/trivy-action` for HIGH and CRITICAL findings), and native plan-mode unit tests (`terraform test` executing `.tftest.hcl` suites, PC-IAC-018).
4. **Speculative Plan against Floci AWS Emulator (`floci-plan`)**: Executes a credential-free speculative plan against the pinned LocalStack Floci service container on port 4566 (`floci/floci@sha256:5ee85db604c4aa74ac76e07ea5cc83086f37bb472bf3dbb97e34019e48efae9c`, PC-IAC-028). An ephemeral S3 bucket is created dynamically via `aws s3 mb` to prove remote state initialization (`use_path_style=true`, `use_lockfile=true`). Strictly plan-only; `apply` and `destroy` are forbidden.
5. **Automated Cost Estimation (`infracost`)**: Downloads checksum-pinned Infracost CLI v2 to estimate monthly costs before (`HEAD~1`) and after (PR head SHA) across all stacks, publishing the dollar and percentage delta table to the job summary (`pc-iac-027.md`). If the secret token or metadata egress approval is missing, it explicitly outputs "estimate unavailable" rather than asserting zero cost.
6. **Peer Review & Immutable Release Tagging**: The designated cross-provider reviewer (`aroma-reviewer-claude` or role SR1) inspects green checks, reviews plan destroys/replacements, approves, and squash-merges into `main`. For reusable modules in `aroma-pacifico-terraform-modules`, an immutable SemVer release tag `vX.Y.Z` (e.g. `v1.2.0`) is published (PC-IAC-015), allowing downstream root stacks to pin exact git references without credentials.

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `standards/iac-rules/pc-iac-015.md` | Entire rule | Reusable module SemVer tagging (`vX.Y.Z`), tag immutability, and public repository reference rules. |
| `standards/iac-rules/pc-iac-016.md` | Entire rule | Zero static secrets, `sensitive = true` variables, and pre-commit secret scanning. |
| `standards/iac-rules/pc-iac-018.md` | Entire rule | Plan-mode unit testing with native `.tftest.hcl` suites. |
| `standards/iac-rules/pc-iac-027.md` | Entire rule | Automated Infracost v2 cost estimation, CLI checksum pinning, token handling, and step summary publication. |
| `standards/iac-rules/pc-iac-028.md` | Entire rule | Floci AWS emulator container (port 4566), test credentials, and speculative plan-only execution. |
| `aroma-pacifico-infrastructure/.github/workflows/` | `infrastructure-ci.yml`, `floci-plan.yml`, `infracost.yml`, `pr-conventions.yml` | Workflow orchestration, job matrices, caching, and gate commands. |
| `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` | Remote state, CI and CD constraints | S3 remote backend, OIDC federation roles, and automated pull request gates. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D01` | GitHub Repositories & Workflows | Source control and CI automation | Feature branch PR workflow, conventional commit enforcement, and matrix gate orchestration. |
| `D11` | CI Pipeline & Plan Gates | Automated reviewable plan generation | Speculative plan execution against the Floci AWS emulator (`terraform plan -input=false -detailed-exitcode`). |
| `D13` | Plan Validation & Evidence | Multi-tool gate verification | Attestation of formatting, validation, tflint, Trivy security scan, and Infracost delta in PR summary. |

---

## Infrastructure as Code CD Promotion and Production Provisioning Sequence

- Source diagram: [`iac-provisioning-sequence.drawio`](iac-provisioning-sequence.drawio)
- Vector publication export: [`iac-provisioning-sequence.svg`](iac-provisioning-sequence.svg)
- Bitmap preview export: [`iac-provisioning-sequence.png`](iac-provisioning-sequence.png)

### Architectural Overview

The Infrastructure as Code Continuous Delivery and provisioning sequence models multi-environment promotion across `dev`, `qa`, and `pdn`, incorporating mandatory manual production approvals, change-window enforcement, and strict dependency-ordered stack deployment.

Promotion proceeds sequentially through three isolated environments:
1. **Plan-Only Guard & Promotion Scope**: Triggered on push to `main` touching infrastructure code or manual workflow dispatch. The `guard` job executes a strict regex check across all workflows and scripts to fail the run if any state-changing command (`terraform apply` or `terraform destroy`) is present (AGENTS.md section 4, ASM-320).
2. **Development & Quality Assurance Stages (`plan-dev` & `plan-qa`)**: Evaluates the four pattern stacks against the Floci emulator in strict dependency order:
   - Stack 1: `stacks/state-backend` (S3 bucket, native S3 locking with `use_lockfile`, KMS `state` CMK, Box D02).
   - Stack 2: `stacks/network` (VPC, public/private subnets, Transit Gateway attachments, route tables, and regional Collection Center association, Box D04/D05).
   - Stack 3: `stacks/security` (GitHub Actions OIDC IAM pipeline roles, Cognito User Pools for staff and field, KMS CMKs, AWS WAF ACLs, Box D06).
   - Stack 4: `stacks/workload` (DynamoDB transactional and telemetry tables, IoT topic rules, EventBridge pipes, SQS queues, Lambdas, Bedrock access, Box D10).
   The `qa` stage serves as the pattern environment (D04, D05, D08) and generates an Infracost cost delta report.
3. **Production Promotion Run 1 (`review-pdn`)**: Initiated by a named Requester (JR1/JR2/SR1) providing `release_note_url` (an HTTPS link under `release-notes/`, ASM-180) or an `incident_reference`. Generates the `pdn` plan across all four stacks, prices monthly cost impact, checks the change window (`scripts/cd-window-check.sh` against the 05:00–18:00 Colombia harvest blackout, ASM-007, ASM-178), computes the SHA-256 digest of the plan manifest (`SHA256SUMS`), outputs the "How to approve" table, and stores the `review-context-pdn` artifact for 7 days.
4. **Production Approval Run 2 (`pdn-approval`)**: Implements a verifiable two-step workflow approval mechanism on GitHub Free (PD-020, `docs/cd-pipeline.md` section 1). A different named Approver listed in the repository variable `PDN_APPROVERS` (SR1 or SR2, enforcing Four-Eyes separation of duties, ASM-172) dispatches Run 2 with `approve_run_id` and `approve_digest`. The `pdn-approval` job runs `scripts/cd-verify-approval.sh`, which confirms approver authorization, validates that Run 1 succeeded on `main` at the exact same commit, downloads artifacts, verifies the SHA-256 digest matches byte-for-byte, re-verifies the change window, and writes the immutable `approval-record-pdn.json` artifact (retained for 90 days).
5. **Production Provisioning Execution**: In a live cloud deployment (shown as the proposed procedure, as the academic challenge stops at `terraform plan`), the approved plan `pdn.tfplan` would be applied in exact dependency order: `state-backend` -> `network` -> `security` -> `workload`. A deployment event is emitted to CloudWatch and the SNS release topic, and the change owner archives the approval record in `release-notes/` within one business day.

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `docs/cd-pipeline.md` | Sections 1–5 | Two-step production approval on GitHub Free (PD-020), plan digest binding, approver verification script, change window check, and 90-day approval record. |
| `docs/automation-and-pipelines.md` | Sections 3.1–3.3, 4.1–4.4, 8, 9 | Repositories, promotion stages, approver roles (ASM-172), Federation notice lead time (ASM-173), and harvest blackout windows (ASM-007, ASM-178). |
| `docs/architecture.md` | Section 14 | Component inventory and authoritative boxes D01–D13 catalog. |
| `docs/decision-log.md` | ASM-007, ASM-172, ASM-173, ASM-178, ASM-180, ASM-320 | Harvest blackout hours, Four-Eyes approver matrix, 2-day notice, release note binding, and plan-only CD boundary. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D01` | GitHub Repositories & Workflows | Orchestration and promotion pipelines | `cd-promotion.yml` and reusable `cd-plan.yml` multi-environment pipeline. |
| `D02` | Remote State Backend | State persistence foundation | `stacks/state-backend` S3 bucket with native locking and KMS `state` CMK. |
| `D04` | Network Architecture | Addressing and connectivity routing | `stacks/network` VPC, subnets, route tables, and Transit Gateway attachments. |
| `D06` | Identity & Security Boundaries | Authentication and execution roles | `stacks/security` OIDC pipeline IAM roles, Cognito User Pools, and WAF Web ACLs. |
| `D12` | Deployment Pipeline Stages | Multi-environment promotion control | Sequential promotion (`dev` -> `qa` -> `pdn`) with automated guard checks. |
| `D13` | Plan Validation & Evidence | Attestation and approval binding | Two-step approval binding named humans to SHA-256 plan digest with 90-day retention. |

---

## Infrastructure as Code Teardown and Rollback Sequence

- Source diagram: [`iac-teardown-and-rollback-sequence.drawio`](iac-teardown-and-rollback-sequence.drawio)
- Vector publication export: [`iac-teardown-and-rollback-sequence.svg`](iac-teardown-and-rollback-sequence.svg)
- Bitmap preview export: [`iac-teardown-and-rollback-sequence.png`](iac-teardown-and-rollback-sequence.png)

### Architectural Overview

The Infrastructure as Code Teardown and Rollback sequence details the operational and disaster recovery protocols for reverting changes and safely decommissioning environment stacks while safeguarding critical historical data.

The sequence models two distinct operational procedures:
1. **Infrastructure Rollback by Reverting and Re-Promoting**: When a defect or regression is detected in production, an authorized approver (SR1 or SR2) decides to roll back and records an incident reference (`INC-YYYYMMDD-NN`, ASM-183). An operator checks out a revert branch and runs `git revert <commit-sha>`. The revert commit is opened as a pull request, verified through PR CI gates, and squash-merged to `main`. The `cd-promotion.yml` pipeline is triggered, re-running promotion mode across `dev`, `qa`, and `pdn`. The speculative plan of the revert displays the exact inverse of the original change (destroys and replacements displayed first in prose). Upon approval in Run 2, the reviewed revert plan is applied, returning infrastructure to the previous clean configuration within the design target of 2 hours (inside the 4-hour lookup RTO, ASM-157, ASM-174). The release note status is updated to `rolled_back` and transmitted to the Federation.
2. **Data Retention Compliance & Safety Guards**: Before any environment teardown or resource destruction can commence, a pre-teardown compliance check verifies three non-negotiable safety guards:
   - **5-Year Telemetry Data Retention Guard**: The DynamoDB `telemetry` table is protected by deletion protection and Point-in-Time Recovery (PITR); long-term S3 telemetry and drone flight archives are protected by S3 Object Lock in Compliance Mode, rendering records non-deletable for 5 years (ADR-003, ASM-070).
   - **5-Year Central Audit Archive Guard**: The centralized S3 audit bucket in the Security Account stores CloudTrail trails, SCP logs, and `approval-record-pdn.json` records under the KMS `audit` CMK with Object Lock Compliance Mode, protected by Service Control Policies against modification or deletion by any principal (ASM-098).
   - **State Backend Safeguard**: The Terraform remote state bucket (`stacks/state-backend`) is **NEVER destroyed first**. It must persist throughout the entire teardown procedure to accurately track, record, and verify the destruction of dependent infrastructure resources.
3. **Controlled Teardown in Reverse Stack Dependency Order**: Decommissioning proceeds strictly in the reverse of provisioning order:
   - Stage 1: `stacks/workload` is destroyed first, terminating Lambda event processors, EventBridge pipes, SQS threshold queues, and IoT topic rules to stop incoming traffic, while preserving locked DynamoDB and S3 data stores.
   - Stage 2: `stacks/security` is destroyed second, removing application IAM roles, Cognito app clients, and WAF associations, while preserving central audit identities and scheduling KMS CMK deletion under a mandatory 30-day waiting period (ASM-097).
   - Stage 3: `stacks/network` is destroyed third, dismantling Collection Center VPN tunnels, Transit Gateway attachments, route tables, subnets, and VPC CIDR blocks after all endpoint dependencies are cleared.
   - Stage 4: `stacks/state-backend` is retired only after all dependent stacks are destroyed and remote state confirms zero active managed resources. The final state file is snapshotted into the central audit archive, and the S3 state bucket is transitioned to an archived or locked hold state.

> [!NOTE]
> **Proposed Procedure Note**: Because the Aroma Pacífico academic delivery is strictly plan-only (`AGENTS.md` section 4, PC-IAC-028), no environment holds live cloud deployments and teardown commands are never run against AWS accounts. The teardown procedure depicted here represents the authoritative **proposed procedure** for lifecycle decommissioning and environment retirement.

### Source Documents and Citations

| Document | Sections | Key Concepts & Decisions |
|---|---|---|
| `docs/cd-pipeline.md` | Section 7 | How to roll back (revert commit, inverted plan, notify, state recovery) and state bucket protection. |
| `docs/automation-and-pipelines.md` | Sections 10, 11 | Rollback mechanisms, time targets (ASM-174), and "cannot be rolled back" compensation table. |
| `docs/architecture.md` | Sections 4, 7, 14 | Telemetry data retention, audit compliance archives, and component inventory D01–D13. |
| `docs/decision-log.md` | ASM-070, ASM-097, ASM-098, ASM-157, ASM-174, ASM-183, ASM-184, ASM-320 | 5-year retention, KMS 30-day deletion window, RTO/RPO targets, revert promotion targets, incident declaration, and plan-only CD boundary. |

### Boxes D01–D13 Traceability

| Box ID | Component Name | Architectural Role | Implementation Pattern |
|---|---|---|---|
| `D01` | GitHub Repositories & Workflows | Rollback version control | Revert PR workflow and re-promotion orchestration via `cd-promotion.yml`. |
| `D02` | Remote State Backend | State integrity safeguard | S3 state bucket persistence throughout teardown; state is never destroyed first. |
| `D04` | Network Architecture | Networking decommission | Controlled teardown of TGW attachments, subnets, route tables, and VPC after workload removal. |
| `D06` | Identity & Security Boundaries | Security decommission | Deletion of application IAM roles and WAF rules while preserving audit keys under 30-day waiting period. |
| `D12` | Deployment Pipeline Stages | Revert re-promotion pipeline | Re-execution of `dev` -> `qa` -> `pdn` promotion with inverted speculative plan review. |
| `D13` | Plan Validation & Evidence | Rollback approval record | Mandatory SR1/SR2 approval and digest verification of the revert plan before execution. |

