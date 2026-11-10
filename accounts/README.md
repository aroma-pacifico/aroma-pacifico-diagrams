# Accounts Diagrams

This folder contains the architectural diagrams for the Aroma Pacífico cloud operational baseline, multi-account structure, governance, identity, budgets, and auditing.

## Landing Zone, Controls, Identity, Budgets and Auditing

- Source diagram: [`landing-zone.drawio`](landing-zone.drawio)
- Vector export: [`landing-zone.svg`](landing-zone.svg)
- Bitmap preview: [`landing-zone.png`](landing-zone.png)

### What it shows

1. **AWS Organization Tree and Accounts**:
   - Organization `aroma-pacifico` in Region US East (N. Virginia) `us-east-1` with all features enabled (ASM-060, ASM-061).
   - Six AWS accounts:
     - `Management Account`: Root level, outside OUs. Consolidated billing, AWS Budgets, Cost Allocation Tags, AWS Cost Categories, AWS Cost Anomaly Detection, IAM Identity Center organization instance. Holds no workloads or application state (ASM-060).
     - `Security Account` (Security OU): Delegated administrator for AWS CloudTrail and IAM Identity Center. Holds central audit archive S3 bucket with Object Lock in compliance mode (5-year retention, ASM-067), customer-managed KMS audit key, Amazon Data Firehose cross-account log ingestion, and Amazon Athena audit query workspace (ASM-061).
     - `Network Account` (Infrastructure OU): Houses the collection-center hub / AWS Transit Gateway (TGW) terminating WAN links for Cauca (10.41.0.0/16), Huila (10.42.0.0/16), and Tolima (10.43.0.0/16), environment VPC attachments, and holds the 10.83.0.0/16 reserve block (ASM-033, ASM-034).
     - `Production Account` (`pdn`, 10.82.0.0/16, Production OU): Pattern production environment with production VPC, S3 state bucket with native locking, GitHub OIDC apply role, Cognito user pools (`staff` and `field`), AWS IoT Core device registry, telemetry DynamoDB store, S3 drone object bucket, and application audit log group (ASM-006).
     - `Test Account` (`qa`, 10.81.0.0/16, NonProduction OU): Pattern test environment with QA VPC, S3 state bucket, GitHub OIDC role, and local test logs (not archived, ASM-068).
     - `Development Account` (`dev`, 10.80.0.0/16, NonProduction OU): Development sandbox with Dev VPC, S3 state bucket, GitHub OIDC role, and local dev logs (not archived, ASM-068).
   - Azure boundary marked "prepared, not deployed" (10.88.0.0/13, ADR-001) as an export target and disaster-recovery contingency without duplicate platform spend.

2. **Controls Applied from the Root**:
   - `Tag Policy` (attached to Root): Enforces PascalCase keys across all pilot resources; enforces allowed values for `Environment` (`dev`, `qa`, `pdn`, `shared`), `CostCenter` (`farms`, `drones`, `collection-centers`, `trips`, `shared`), and `CollectionCenter` (`cauca`, `huila`, `tolima`).
   - `Service Control Policies (SCPs)`:
     - Root: Deny leaving the organization (`organizations:LeaveOrganization`).
     - Root: Deny Regions other than `us-east-1` (`aws:RequestedRegion != us-east-1`, exempting global services).
     - Root: Protect the audit trail (denies `cloudtrail:StopLogging`, `DeleteTrail`, `UpdateTrail`, `PutEventSelectors`; protects audit archive bucket policy, Object Lock compliance, and KMS key deletion/disabling).
     - Root: Deny long-term IAM credentials (`iam:CreateUser`, `CreateAccessKey`, `CreateLoginProfile`).
     - Root: Deny root user of member accounts (centralized root access management).
     - Workloads OU: Require cost tags on creation (`Environment`, `CostCenter`).
     - Production OU: Production changes only through the CI/CD pipeline (denies human IAM Identity Center write actions in `pdn`; break-glass and narrow ops exempt under incident reference).

3. **Identity Entry Points (Exactly 4 Kinds)**:
   - *Kind 1 (AWS Human Operators)*: IAM Identity Center (organization instance in Management Account, delegated administrator in Security Account). Mandatory MFA via TOTP software token. Permission sets per role (Auditor, Platform Operator, Break-Glass). Zero long-term IAM users or static keys.
   - *Kind 2 (CI/CD Pipeline)*: GitHub OIDC Identity Provider in each workload account (`dev`, `qa`, `pdn`). Short-lived AWS STS credentials with per-environment plan and apply IAM roles (ADR-003).
   - *Kind 3 (Hardware Devices)*: AWS IoT Core Thing Registry in each workload account. Mutual TLS (mTLS) with unique X.509 client certificate registered from CSR (port 8883 / 443 ALPN). Immediate certificate revocation.
   - *Kind 4 (Application Users)*: Amazon Cognito User Pools in `pdn`:
     - `staff` pool (MFA required with TOTP): Collection-center operator, drone operator, agronomist, platform operator. Sensitive operations require fresh auth <= 10 min (ASM-092).
     - `field` pool (No MFA, justified adoption decision ASM-091): Farmer, transporter, read-only buyer. Passwordless SMS one-time code, 30-day refresh token, hardware-backed keystore offline outbox encryption (ASM-095).
     - Anonymous Public Reader: Unauthenticated sack lookup by code only; no Cognito user pool (ASM-069). Zero cédula or full trace returned.

4. **Budgets and Cost Alerts Path**:
   - 11 monthly AWS Budgets in Management Account: 5 cost line budgets (`farms`, `drones`, `collection-centers`, `trips`, `testing`) and 6 per-account budgets.
   - Alert thresholds at actual 80%, actual 100%, and forecasted 100% (ASM-066).
   - Alert notification path: AWS Budgets and AWS Cost Anomaly Detection monitors trigger notifications via customer-managed KMS-encrypted Amazon SNS topic to Platform Operator and Cost Owner.
   - Testing Budget Action: Automated guardrail at actual 100% test spend applies SCP to NonProduction OU denying creation of new resources (requires manual approval).
   - Production Protection Rule: Production OU never receives an automatic budget stop action (protects 4-hour RTO for alerts and public lookup).

5. **Audit Pipeline for the Four Mandatory Events**:
   - Dual-layer audit model: AWS CloudTrail for AWS API calls; Application and IoT Core layers for domain business events. All archived for 5 years in Security Account under S3 Object Lock compliance mode (ASM-067).
   - *Event 1 (Sack Read)*: Written by application lookup service to `/aws/application/audit` CloudWatch Logs in `pdn`. Shipped via cross-account subscription filter to Amazon Data Firehose in Security Account to `s3://aroma-audit-archive-shared/application/`.
   - *Event 2 (Threshold Change)*: Written by application threshold service to `/aws/application/audit` in `pdn` in same transaction. Shipped via Firehose to `s3://.../application/`. Secondary: CloudTrail write-only data events on `thresholds` DynamoDB table.
   - *Event 3 (Telemetry Publication)*: Accepted MQTT readings written by IoT topic rule with metadata (node ID, cert principal, client ID, source IP, receive time, original timestamp) to DynamoDB telemetry store (5-year retention). Rejected connection/publish logged to `AWSIotLogsV2` at ERROR level.
   - *Event 4 (Flight Object Delivery)*: CloudTrail S3 data events (`PutObject`, `CompleteMultipartUpload`) on `pdn` drone bucket by ARN prefix delivered via Multi-Region Organization Trail to `s3://.../AWSLogs/<org-id>/`. Reception function links S3 object to flight record in transactional store (ADR-002).
   - Auditor access: Auditor role queries both audit streams via Amazon Athena workspace in Security Account.

6. **Tagging Scheme Legend and Traceability**:
   - Full specification table for 8 transversal and pilot tags: `Client`, `Project`, `Environment`, `Owner`, `CostCenter`, `FarmId`, `CollectionCenter`, `Name`.
   - Activation of cost allocation tags: `Environment`, `CostCenter`, `CollectionCenter`, `FarmId`, `Project`.
   - Shared managed services allocation formula based on stored usage counts (ASM-065).

### Source documents

- `docs/landing-zone.md` (all sections): Organization structure, root controls, identity entry points, tagging scheme, budgets and cost alerts, audit of the four mandatory events, alternatives considered, and decision-log citations.
- `docs/security-and-identity.md` (section 4): Roles, least privilege, separation of duties, MFA requirements, and pool separation (`staff` vs `field`).
- `docs/architecture.md` (section 14): Diagram boxes D01-D13 catalog and traceability to IaC.
- `docs/decision-log.md`: ASM-060 to ASM-069, PD-070 to PD-079, ADR-001, ADR-002, ADR-003.

### Boxes D01-D13 shown

- `D02`: One S3 state bucket per environment with S3 native locking (State bucket (pdn), State bucket (qa), State bucket (dev); ADR-003).
- `D03`: Per-environment configuration, secrets management, and transversal tags (`common_tags`).
- `D04`: VPC boundaries, tiers, and private endpoints of pattern environment (`pdn`, `qa`, `dev`).
- `D06`: Identities (IAM Identity Center, Cognito User Pools, GitHub OIDC federation), customer-managed KMS keys, and access controls.
- `D09`: Telemetry ingestion pattern module (AWS IoT Core device certificates, topic rules, DynamoDB telemetry store).
