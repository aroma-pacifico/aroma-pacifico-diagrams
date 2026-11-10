# Pipeline Diagrams

This folder contains the architectural and flow diagrams for the Aroma Pacífico continuous integration, continuous delivery, telemetry ingestion, and flight-object reception pipelines.

## Platform CI/CD Pipeline and Release Model

- Source diagram: [`platform-pipeline.drawio`](platform-pipeline.drawio)
- Vector export: [`platform-pipeline.svg`](platform-pipeline.svg)
- Bitmap preview: [`platform-pipeline.png`](platform-pipeline.png)

### What it shows

1. **Source Repositories and Triggering Paths (A18 section 3.1)**:
   - `aroma-pacifico-terraform-modules` (public): Contains reusable infrastructure modules under `modules/<name>/**`. Produces SemVer git tags `?ref=vX.Y.Z`. Module CI verification gates run `terraform fmt -check -recursive`, `terraform validate`, `tflint`, Trivy/Checkov scanners, and mock-provider `terraform test`. Strictly zero cloud credentials or account IDs are stored in this public repository (ADR-003).
   - `aroma-pacifico-infrastructure` (private, D01): Contains root stacks per environment and domain (`stacks/network/**`, `stacks/security/**`, `stacks/state-backend/**`, `stacks/workload/**`) and `stacks/workload/releases.tfvars` (ASM-171). Infrastructure pull requests trigger module gates, secret scanning, verification that all digests in `releases.tfvars` exist in all three artifact buckets, and generate saved plans for `dev`, `qa`, and `pdn` attached to the pull request with destroys and replacements listed first.
   - `aroma-pacifico-platform` (private, future application repository, ASM-170, PD-170): Houses application APIs, services, Lambda functions, and web frontends. Pull requests run unit tests, linting, secret scanning, vulnerability scanning, package build checks, and backward-compatibility contract tests against current and previous schemas (ASM-175). Tag pushes build immutable packages, compute SHA-256 digests and SBOMs.

2. **Pipeline Identity and Security (ADR-003, A18 section 3.3)**:
   - GitHub Actions OIDC identity provider (`token.actions.githubusercontent.com`) authenticates jobs directly into short-lived AWS STS credentials with zero static IAM access keys or stored credentials (A08 SCP "Deny long-term credentials").
   - Three dedicated IAM roles:
     - *Plan Role (ADR-003)*: Trusted subject bound to pull requests and `main` tokens of `aroma-pacifico-infrastructure`. Rights strictly limited to reading state, acquiring locks (`*.tflock`), and read-only describe API actions.
     - *Apply Role (ADR-003)*: Trusted subject bound to GitHub deployment environments (`dev`, `qa`, `pdn`). Rights granted to apply the saved plan and execute CodeDeploy deployments.
     - *Artifact Publisher Role (ASM-171)*: Trusted subject bound to release tag pushes of `aroma-pacifico-platform`. Rights restricted to `s3:PutObject` on `artifacts/*` in the environment artifact buckets, with zero data read access, zero Lambda permissions, and zero CodeDeploy access.
   - Remote State Backend (D02): Dedicated encrypted S3 bucket per environment (`aroma-dev-tfstate`, `aroma-qa-tfstate`, `aroma-pdn-tfstate`) with native S3 locking (`*.tflock`), customer-managed KMS key (`state` key), versioning, and `prevent_destroy`.
   - Secrets Management (D03, A18 section 11): Secrets kept strictly out of repositories and out of state. AWS Secrets Manager and SSM Parameter Store (`SecureString`) read ephemerally into write-only arguments. Lambda roles fetch credentials at runtime rather than via environment variables.

3. **Common Release Model and Promotion by Construction (ASM-171)**:
   - Release Candidate (RC): Defined as exactly one merge commit SHA on `main` of `aroma-pacifico-infrastructure`. Pins module tags (`?ref=vX.Y.Z`) and pins version and SHA-256 digests in `stacks/workload/releases.tfvars`. Because the identical commit is planned and applied across `dev`, then `qa`, then `pdn`, the identical verified artifact is promoted by construction.
   - Immutable Artifact Staging: Platform builds stage packages once into the environment artifact S3 bucket (`artifacts/functions/<name>/<semver>/<sha256>.zip`). Static web bundles (reports web, lookup page) are unpacked to `releases/<semver>/` in the assets bucket; the active release is switched via CloudFront origin path in `releases.tfvars`, requiring zero cache invalidation (ASM-176).
   - Function Deployment Pattern (A18 section 4.3): Terraform declares Lambda functions with `publish = true` and declares the `live` alias and CodeDeploy application. Event sources target the `live` alias ARN, never `$LATEST`. The `live` alias carries the lifecycle rule `ignore_changes = [function_version, routing_config]` as permitted by PC-IAC-010 section 3.2 for external deployment agents. CodeDeploy shifts traffic and executes automatic rollback.

4. **Environment Stages, Gates, and Approvals (A18 section 4, section 8)**:
   - `dev` stage (Development Account, 10.80.0.0/16, NonProduction OU): Integration of every merge commit to `main`. Apply saved plan via apply role; CodeDeploy `LambdaAllAtOnce`. No approval required; errors fixed forward. Local test logs not archived to audit (ASM-068).
   - `qa` stage (Test Account, 10.81.0.0/16, NonProduction OU): The pattern environment evaluated by Epic 6 (ASM-051). Requires `dev` success for the same commit and approval by 1 reviewer from SR1, SR2, JR1, JR2 (other than the author). CodeDeploy runs `LambdaCanary10Percent5Minutes` (identical to `pdn` to exercise canary and rollback before production, ASM-174). Executes formal acceptance suite (ASM-177): API contract tests, sack lifecycle replay, lookup buyer view checks (ASM-096), staff MFA sign-in, synthetic telemetry, and recorded drone mission upload. Results recorded in GitHub deployment status.
   - `pdn` stage (Production Account, 10.82.0.0/16, Production OU): Changes strictly restricted to this pipeline (A08 SCP). Four gates evaluated cheapest first:
     1. QA acceptance succeeded for exact same commit SHA.
     2. Release note exists and notice sent to Federation at least 2 business days before window (ASM-173, Q-15).
     3. Harvest-week window check: strictly blocked 05:00 - 18:00 Colombia time on harvest days in Cauca, Huila or Tolima (ASM-007, ASM-055); expected end (apply + canaries + bake) must complete before 05:00 (ASM-178). Incident exception only with `INC-YYYYMMDD-NN` reference (ASM-183).
     4. Required reviewers: SR1 or SR2, other than author and other than person who triggered the run (ASM-172).
     Apply saved plan via apply role; CodeDeploy `LambdaCanary10Percent5Minutes`. Automatic rollback on CloudWatch alarm set (`<prefix>-pdn-alarm-<function>-errors`, throttles, DLQ age, API Gateway 5XX) within 10 minutes.

5. **Rollback Mechanisms and Forward Correction (A18 section 10)**:
   - Four rollback mechanisms: CodeDeploy automatic rollback (< 10 min), rollback deployment moving alias (< 30 min), Git revert commit through promotion (< 2 hr, inside 4-hour lookup RTO ASM-157), and dead-letter queue redrive by platform operator.
   - Forward correction model for non-rollbackable state:
     - Sack events: `sack_event_log` is immutable (conditional puts only; no update/delete). "Un despliegue no pierde eventos de saco" holds. Defective events corrected via compensating events (ASM-151); derived tables rebuilt by sack applier (A16 section 9.2).
     - Stored telemetry: Audited batch correction job updates items or marks them unreliable in history.
     - Sent alerts: Correction notice dispatched over same channels.
     - Inference results: Superseded by newer result tagged with `function_version` (ASM-175).
     - Quarantined / restricted objects: Restored from versioned S3 under incident reference.

6. **Release Notes, Federation Notice, and Plan-Only Mandate (A18 section 12, A22)**:
   - Release notes templates: Application template for user-visible changes and Infrastructure template for network, firewall, ingestion, and drone path (ASM-181).
   - Federation notice: Dispatched >= 2 business days before window; outcome and rollbacks published via `pdn` release SNS topic (ASM-182).
   - Plan-only boundary: Academic DevOps challenge never runs `terraform apply`, never creates deployments, and never publishes artifacts to cloud accounts. All verification is evidenced via scanners, mock tests, and plan outputs (ADR-003, B13).

### Source documents

- `docs/automation-and-pipelines.md` (all sections): Scope, repositories, environments, pipeline identities, release model, gates, CodeDeploy pattern, promotion procedure, harvest-week window, approval roles, Federation notice, rollback targets, secrets, and observability hooks.
- `docs/release-notes/README.md` (all sections): Application and infrastructure templates, window restrictions, incident exceptions, and field mapping from pipeline events.
- `docs/architecture.md` (sections 6, 10, 14): Automation and change, component inventory, diagram boxes catalog D01 to D13.
- `docs/decision-log.md`: ADR-001, ADR-003, ASM-007, ASM-008, ASM-051, ASM-055, ASM-068, ASM-151, ASM-157, ASM-170 to ASM-179, ASM-180 to ASM-185, PD-019, PD-020, PD-070, PD-170 to PD-176.

### Boxes D01-D13 shown

- `D01`: `aroma-pacifico-infrastructure` with one root per environment, consuming modules by tag.
- `D02`: One S3 state bucket per environment with native locking.
- `D03`: Per-environment configuration and secrets, out of the repository and the state.
- `D11`: CI pipeline producing an executable, reviewable plan.
- `D12`: CD pipeline with `pdn` approval and the harvest-week window.
- `D13`: Final validation and evidence of the executable plan.

---

## Ingestion and Flight-Object Reception Pipelines

- Source diagram: [`ingestion-and-flight-objects.drawio`](ingestion-and-flight-objects.drawio)
- Vector export: [`ingestion-and-flight-objects.svg`](ingestion-and-flight-objects.svg)
- Bitmap preview: [`ingestion-and-flight-objects.png`](ingestion-and-flight-objects.png)

### What it shows

1. **Telemetry Ingestion Pipeline (Left-to-Right Flow)**:
   - *1. Field Sensor Nodes and Bearers*: Two node classes (crop node publishing 6 readings/day, drying node publishing 24 readings/day while active). Communication via MQTT over TLS 1.2+ on port 8883 (or port 443 ALPN) with unique per-node X.509 client certificates registered from CSRs (mTLS, D06). No static keys. Encrypted non-volatile ring buffer holds >= 24 hours of readings; upon reconnect, publishes live reading first, then drains oldest first (released by `PUBACK`). LTE multi-network SIM bearer; satellite IP bearer on farms out of coverage (ASM-073). Direct connection to regional AWS IoT Core endpoint bypassing collection centers and VPC subnets (CL-TEL-04).
   - *2. Regional Ingestion and Topic Rule (B09, D09)*: AWS IoT Core public ATS endpoint in `us-east-1` validates client certificates against Thing Registry. Topic rule matches `aroma/telemetry/#` and appends audit metadata (node ID, certificate principal ARN, client ID, source IP, receive time, measurement timestamp). Direct DynamoDB action writes reading to `telemetry` table. SQS action enqueues reading reference to `threshold_queue`. Rule error action routes failed deliveries to SQS error queue (`aroma-telemetry-error-queue`) for redrive (ASM-161). Connection/publish failures logged to `AWSIotLogsV2` at ERROR level.
   - *3. Primary Storage Target*: Amazon DynamoDB `telemetry` table in on-demand mode. Partition key `node_id`, sort key `timestamp`. Encrypted with customer-managed KMS key (`telemetry` key). Deletion protection (`prevent_destroy = true`) and Point-in-Time Recovery (PITR) enabled. Five-year retention enforced by TTL attribute. Stores audited event 3 metadata. Accommodates 3,600 readings/day peak across 200 farms and 3,600 readings/min drain surges; scalable to 1,000 farms (18,000 readings/day).
   - *4. Threshold Evaluation and Alert Dispatch (A14, A17)*: Amazon SQS `threshold_queue` buffers incoming readings. Lambda `threshold` function (`live` alias, batch size 10) evaluates readings against versioned `thresholds` DynamoDB table using the threshold version in force at the reading's *original timestamp* (ASM-138). Idempotent conditional writes prevent duplicate evaluations. Evaluates once per episode (node, variable, condition), not per reading; opens episode, persists `crop_alert_raised` in `cases` table, and switches plot to precision mode. Publishes to KMS-encrypted Amazon SNS `alert_topic`; Lambda `notification_dispatcher` sends dual-channel notification via app push (when data connection is active) and SMS (direct cellular signal without data, ASM-136), with a delivery target of < 5 minutes for live readings (ASM-075).
   - *5. Failure Paths, Alarms, and Rollback (A18 section 6, section 10)*: SQS `threshold_dlq` captures unhandled failures after 5 attempts with backoff; operator redrives without redeployment. CloudWatch alarms monitor alias errors, throttles, DLQ message age, and `ReadingNotPersisted`. CodeDeploy canary rollback reverts alias to previous version (< 10 min). Infrastructure changes rolled back via Git revert commits (< 2 hr). Forward correction handles stored readings with wrong attributes via audited batch correction jobs, sent alerts via correction notices, and revoked certificates via re-enrollment with new CSRs.

2. **Flight-Object Reception Pipeline (Left-to-Right Flow)**:
   - *1. Ground Station Upload and Ingress (A15, D04)*: Contracted certified commercial drone operator operates ground station from collection center Soporte zone (IP 10.4x.2.10, VLAN 102) or municipal seat (ADR-002). Upload authorization via 15-minute S3 pre-signed upload URLs generated by API role (ASM-093) with SHA-256 checksums. Collection center ingress routes through center firewall, IPsec VPN tunnel, AWS Transit Gateway (D05), and private S3 interface endpoint in the `object_storage` tier (D04). The `drone_object_ingress` tier is RESERVED (ASM-140) and not used as an active ingress path. Operator encrypted local buffer deletes files only upon receiving platform persistence receipt (A15 section 7).
   - *2. S3 Drone Object Bucket and Notifications (D06)*: Target bucket `aroma-<env>-drone-objects` in `us-east-1`. S3 Block Public Access enabled across all four dimensions; S3 Object Versioning enabled; customer-managed KMS key (`objects` key); 5-year retention enforced by lifecycle rules. Two-phase upload: flight objects uploaded first (orthomosaics, multispectral photos, GPX tracks, logs), followed by flight manifest JSON (`manifest.json`). S3 event notification `s3:ObjectCreated:*` on the manifest invokes Lambda `reception` function on `live` alias. CloudTrail S3 data events (`PutObject`, `CompleteMultipartUpload`) logged to audit archive in Security Account (audited event 4, ASM-067).
   - *3. Reception Function and Screening (A15, A17)*: Lambda `reception` function executes four strict validation gates:
     1. Manifest verification against active upload grant and SHA-256 checksums.
     2. Quarantine gate moving ungranted or invalid checksum objects to `quarantine/` prefix.
     3. Privacy screening (ASM-164, ASM-165) screening images for human presence; objects flagged `person_present` moved to `restricted/person/` prefix encrypted with customer-managed `restricted` KMS key. Only screened crop images can ever leave Colombia to Bedrock.
     4. Persistence receipt emitted to ground station once all objects and manifest are linked (`record_persisted`).
     Updates DynamoDB `missions` table with flight record, mission state (`record_persisted`), and polygon versions (`records` KMS key, 5 years).
   - *4. EventBridge Routing and Bedrock Inference (A17, B10, D10)*: DynamoDB Stream on `missions` table captures `record_persisted`. Amazon EventBridge Pipe filters and routes events to custom `aroma-drone-bus`. Four mission-type EventBridge rules route to dedicated SQS queues:
     1. Delimitation rule -> SQS -> Delimitation Lambda (activates new polygon version in area mode; zero model inference).
     2. Confirmation rule -> `infconfirmqueue` (DLQ `infconfirmdlq`) -> Confirmation Lambda.
     3. Estimation rule -> `infestimatequeue` (DLQ `infestimatedlq`) -> Estimation Lambda.
     4. Verification rule -> `infverificationqueue` (DLQ `infverificationdlq`) -> Verification Lambda.
     Inference functions call vision-capable foundation model (Amazon Nova Pro in `us-east-1`) by model ID. Packages embed zero-shot prompt and output schema (`prompt_version`, ASM-179). Results persisted with timestamp, confidence, `function_version`, and source object into `missions` and `cases` tables. Strict human governance (ADR-002, Part E): Federation agronomist with MFA session <= 10 min authorizes fumigation orders; Bedrock NEVER authorizes fumigation or closes cases automatically.
   - *5. Failure Paths, Alarms, and Recovery (A18 section 7, section 10)*: Per-function DLQs capture failed invocations after 5 attempts with backoff; operator redrives without redeployment. Idempotent execution ensures zero duplicate records on retries. CloudWatch alarms monitor alias errors, throttles, DLQ message age, `InferenceFailed`, and `MissionsAwaitingResult`. CodeDeploy canary rollback reverts alias to previous version (< 10 min). Infrastructure changes rolled back via Git revert commit (< 2 hr). Forward correction ensures uploaded objects are never deleted; inference results are superseded (never edited); wrongly activated polygons replaced by activating previous version as new version; quarantined/restricted objects restored from versioned S3 under incident reference (`INC-YYYYMMDD-NN`).

### Source documents

- `docs/automation-and-pipelines.md`: sections 6 (ingestion pipeline), 7 (flight object reception pipeline), 10 (rollback), and 11 (secrets).
- `docs/farm-monitoring.md`: sections 3 (load), 4 (thresholds), 5 (evaluation), 6 (drying lots), 7 (notifications), and 10 (observability).
- `docs/drone-service.md`: sections 5 (cases and orders), 6 (mission package), 7 (persistence receipt), 9 (ingress endpoints), 10 (no-link behavior), and 12 (observability).
- `docs/inference-functions.md`: sections 2 (managed inference), 3 (triggers), 4 (four functions), 6 (failures), 7 (observability), 8 (Region and privacy), and 11 (IaC pattern).
- `docs/architecture.md`: sections 5 (network and data paths), 7 (workloads), 10 (component inventory), 11 (pest-to-fumigation flow), 12 (store-and-forward), and 14 (box catalog).
- `docs/decision-log.md`: ADR-002, ASM-010, ASM-067, ASM-073, ASM-074, ASM-075, ASM-093, ASM-135, ASM-136, ASM-138, ASM-140, ASM-147, ASM-148, ASM-149, ASM-161, ASM-162, ASM-164, ASM-165, ASM-175, ASM-179, PD-018, PD-071, PD-073, PD-140, PD-161, PD-162, PD-167.

### Boxes D01-D13 shown

- `D04`: VPC endpoints for ground station (`application` execute-api and `object_storage` S3 interface endpoint in collection center).
- `D06`: Identities (IoT X.509 certificates, OIDC roles, upload grant roles) and customer-managed KMS keys (`telemetry`, `objects`, `records`, `restricted`).
- `D09`: Telemetry ingestion pattern module (IoT topic rule, error queue, telemetry table).
- `D10`: Event-driven inference function pattern (EventBridge pipe/bus, SQS queues, DLQs, Lambda inference functions, Bedrock).
