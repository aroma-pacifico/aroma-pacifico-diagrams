# Security Diagrams

This folder contains the architectural and sequence diagrams for the Aroma Pacífico security model, identity directories, authentication protocols, least-privilege role scopes, public sack lookup isolation, personal data lifecycle, cryptographic key management, and device/user revocation.

## Identity, Directory Architecture, Least Privilege and Encryption Keys

- Source diagram: [`identity-and-access.drawio`](identity-and-access.drawio)
- Vector export: [`identity-and-access.svg`](identity-and-access.svg)
- Bitmap preview: [`identity-and-access.png`](identity-and-access.png)

### What it shows

1. **Kinds of Identity (Four Kinds, Never Mixed | Section 2)**:
   - **Kind 1 (Workforce — Humans operating AWS)**: Platform Operator, Auditor, and Break-Glass. Directory: IAM Identity Center (AWS Organizations instance in Management Account, delegated administrator in Security Account). Mandatory MFA on every sign-in via TOTP software token. Zero IAM users, zero static access keys (A08 §3.1). Permission sets: `PlatformReadOnly` (4 hours), `DeviceRevocation` (1 hour), `IdentitySupport` (1 hour), `Auditor` (4 hours), and `BreakGlass` (1 hour, automated alerts). Production infrastructure mutations allowed strictly via CI/CD pipeline.
   - **Kind 2 (Application Users)**: Farmer, transporter, collection-center operator, read-only buyer, drone operator, agronomist, and platform operator. Directory: Amazon Cognito user pools (Essentials plan, `us-east-1`, Workload Account `pdn` [D06]). Structurally separated into two user pools:
     - `staff` pool (MFA Required via TOTP software token): Collection-center operator, drone operator, agronomist, and platform operator. Sensitive staff operations require token `auth_time` <= 10 minutes (ASM-092).
     - `field` pool (No MFA, justified adoption decision ASM-091): Farmer, transporter, and read-only buyer. Passwordless sign-in via SMS OTP (farmer/transporter) or email OTP (buyer).
     - *Kind 2b (Anonymous Public Reader)*: Unauthenticated sack lookup by 80-bit Crockford Base32 code; returns strictly the buyer view projection; zero Cognito pool allocation (ASM-058).
   - **Kind 3 (Field Devices)**: Hardware identities in AWS IoT Core Thing Registry [D08, D09]. Mutual TLS (mTLS, port 8883 / 443 ALPN) with unique X.509 client certificate registered from on-device CSR (ADR-003, ASM-010). Private keys generated on-device, never leave non-volatile hardware storage. Dynamic policy variables `${iot:Connection.Thing.ThingName}`. Drone upload grant protocol binds pilot MFA session to active drone kit certificate (15-minute S3 pre-signed URLs, ASM-093, PD-075).
   - **Kind 4 (CI/CD Pipelines & Platform Services)**: GitHub Actions via GitHub OIDC Identity Provider [D06, D11, D12]. Short-lived AWS STS credentials for scoped plan and apply roles per environment. Platform IAM service roles with least privilege (Inference Lambdas, Threshold Lambda [D10], Reception Lambda, IoT topic rule role [D09], API Gateway execution roles).

2. **Application Entry Points & Network Boundaries (Section 3, Addressing §4 [D04], Sack & Trip §7.2 [PD-051])**:
   - Public Sack Lookup Front Door: Amazon CloudFront CDN + AWS WAF (rate limited <= 30 req/min per IP | PD-102) -> Lambda Function URL via Origin Access Control (OAC). Empty `public_query` tier; zero load balancer, zero NAT gateway charges (PD-051 closed).
   - Application API Front Door: Amazon API Gateway custom domain (HTTPS, TLS 1.2 policy) -> Regional public endpoint for mobile apps; private VPC `execute-api` interface endpoint in `application` tier (10.82.48.0/20) for Collection Center logistics workstations via IPsec TGW VPN [D04, D05].
   - S3 Interface Endpoint: Located in `object_storage` tier (10.82.80.0/20) reachable from collection center Soporte LAN (10.4x.2.10) for drone uploads.
   - Cognito Interface Endpoint: Located in `application` tier (10.82.48.0/20) allowing logistics staff to authenticate during WAN degradation.

3. **Roles, Least Privilege, Scope Enforcement & Separation of Duties (Section 3.3, 4, 8, 9)**:
   - Dynamic Scope Enforcement: Scopes live in transactional database records, NOT inside JWT tokens (`Own farms`, `Own trips`, `Own center`, `Assigned missions`, `Department`). Adding, updating, or revoking scope takes effect on the next API call (0 minutes).
   - Architectural Invariants:
     - Agronomist authorizes fumigation; CANNOT fly missions; no automated trigger authorizes (ADR-002, Case Part E).
     - Drone Operator flies missions; CANNOT create, modify, or authorize orders.
     - Platform Operator manages enrollment/infrastructure/support; CANNOT authorize fumigation; CANNOT write business records (sacks, trips, orders).
     - One Identity = One Role: Cognito pre-authentication Lambda trigger refuses sign-in to users belonging to multiple role groups (ASM-099).
     - Sensitive Staff Operations: Require `auth_time` <= 10 minutes (ASM-092).
   - Personal Data & Privacy Protection:
     - Cédula isolation: Nobody reads another person's cédula except Platform Operator during enrollment/support. Logs carry `subject_id` or `sub`, never cédula.
     - Pseudonymous `subject_id`: All business transactions reference persons via random opaque `subject_id`.
     - 5-year retention clock: Counted from deactivation for profiles, from event date for transactions (ASM-098).
     - Anonymization engine: Monthly EventBridge Scheduler function deletes personal profile in DynamoDB and deletes Cognito user; `subject_id` remains unlinked.

4. **KMS Customer-Managed Keys (CMK) Architecture (Section 7.2, 7.3 [D06])**:
   - 8 Environment CMKs in each environment (`pdn`, `qa`, `dev`):
     - `telemetry` [D06, D09]: Encrypts DynamoDB `telemetry` table and transporter `transport_positions` store.
     - `objects` [D06]: Encrypts S3 Drone Object Bucket (imagery, maps, flight records, sack history JSON Lines export).
     - `records` [D06]: Encrypts DynamoDB Transactional Tables (`sacks`, `sack_event_log`, `trips`, `cases`, `orders`, `thresholds`, `upload_grants`).
     - `personal` [D06]: Encrypts DynamoDB `personal_data` table. Strictly denied to inference and lookup roles.
     - `logs` [D06]: Encrypts CloudWatch Log Groups (`/aws/application/audit`, function logs, `AWSIotLogsV2`, VPC Flow Logs).
     - `state` [D02, D06]: Encrypts Terraform Remote State S3 Buckets (`aroma-pdn-tfstate`, etc.).
     - `public_view` [D06]: Encrypts DynamoDB `buyer_view` projection table. Granted ONLY to lookup Lambda and projection updater.
     - `restricted` [D06]: Encrypts S3 prefix `restricted/person/`. Explicit Deny for all function and human roles; incident audit access only.
   - Cross-Cutting Account Keys:
     - `audit` (Security Account CMK | D06): Encrypts Central S3 Audit Archive (5-year Object Lock compliance mode).
     - `budgets_sns` (Management Account CMK | D06): Encrypts SNS topic for AWS Budgets & Anomaly alerts.
   - Key Governance:
     - Automatic 365-day rotation on all CMKs (ASM-097). Previous key material preserved; zero re-encryption required.
     - Deletion protection: `kms:ScheduleKeyDeletion` and `kms:DisableKey` denied to all roles except `BreakGlass`; 30-day waiting period alerts operators.

5. **Encryption in Transit & Edge Protection (Section 7.1, Connectivity §2)**:
   - Zero unencrypted flows: TLS 1.2 or higher end-to-end across all external and internal endpoints.
   - AWS IoT Core: MQTT over TLS, mutual TLS, TLS policy `IoTSecurityPolicy_TLS13_1_2_2022_10` (PD-100).
   - CloudFront & Public Lookup: HTTPS only, minimum security policy `TLSv1.2_2021` (PD-051).
   - API Gateway: Custom domain enforcing HTTPS only, security policy `TLS_1_2`.
   - S3: Bucket policies enforce `aws:SecureTransport: true` and `s3:TlsVersion >= 1.2`.
   - Edge encryption: Farm node NV circular buffer (ASM-074), Mobile app outboxes in Android Keystore / iOS Secure Enclave (ASM-095), Drone ground station encrypted local disk (ASM-095).

### Source documents

- `docs/security-and-identity.md` (all sections): Kinds of identity, Cognito pools, least-privilege matrix, MFA rules, device identities, encryption in transit and at rest, sack code format, buyer view projection, personal data retention, and revocation procedures.
- `docs/landing-zone.md` (section 4): Identity entry points for AWS operators, pipelines, devices, and application users.
- `docs/architecture.md` (section 14): Diagram boxes D01-D13 catalog and traceability to IaC.
- `docs/decision-log.md`: ASM-090 to ASM-099, ADR-001, ADR-002, ADR-003, PD-075, PD-100, PD-102.

### Boxes D01-D13 shown

- `D02`: One S3 remote state bucket per environment with native locking protected by `state` KMS CMK.
- `D04`: VPC boundaries, tiers, and private interface endpoints (`execute-api`, `cognito-idp`, S3, CloudWatch Logs).
- `D06`: Identities (IAM Identity Center, Cognito User Pools, GitHub OIDC roles, IAM service roles), 8 KMS customer-managed keys, and initial access controls.
- `D08`: Farm node device identity (CSR-registered X.509 client certificate in AWS IoT Core).
- `D09`: Telemetry ingestion (AWS IoT Core endpoint, topic rules, DynamoDB telemetry table).
- `D10`: Threshold function pattern consuming SQS threshold queue and writing alert records.
- `D11`: CI pipeline executing plan role via GitHub OIDC federation.
- `D12`: CD pipeline executing apply role via GitHub OIDC federation.

---

## Authentication with MFA, Public Buyer Lookup & Revocation Protocols

- Source diagram: [`revocation-sequence.drawio`](revocation-sequence.drawio)
- Vector export: [`revocation-sequence.svg`](revocation-sequence.svg)
- Bitmap preview: [`revocation-sequence.png`](revocation-sequence.png)

### What it shows

1. **Scenario 1: Authentication Protocols & Sensitive Operation Step-Up MFA (Sections 5.1, 5.2, ASM-091, ASM-092)**:
   - **Part 1A (Staff Sign-In with Mandatory TOTP MFA)**: Staff user submits username and password to Cognito `staff` pool; Cognito issues `SOFTWARE_TOKEN_MFA` challenge; user submits 6-digit TOTP from mobile authenticator app; Cognito validates and returns JWT tokens (Access 60m, ID 60m, Refresh 30d) with `auth_time = NOW`.
   - **Part 1B (Sensitive Staff Operation Step-Up Authentication | ASM-092)**: Agronomist calls `POST /orders/{id}/authorize` with Bearer token. API Lambda inspects `auth_time`:
     - If `(current_time - auth_time) > 10 minutes`, API returns `401 Unauthorized: Step-Up Required (auth_time > 10m)`.
     - App prompts for fresh TOTP code; user enters code; Cognito issues fresh token with `auth_time <= 10m`.
     - Request resubmitted; API validates `auth_time <= 10m` and verifies Separation of Duties (`authorizer_id != operator_id` | ADR-002). Transaction committed to DynamoDB `records` table.
   - **Part 1C (Field User Passwordless Sign-In | Justified Decision ASM-091)**: Farmer/transporter unlocks phone (biometric/PIN); enters phone number; Cognito sends SMS OTP; user enters OTP; Cognito validates and returns JWT (Access 60m, Refresh 30d). App generates hardware-backed keystore outbox encryption key (ASM-095).
   - **Offline Work & Outbox Flushing**: Field records queued in encrypted outbox; upon reconnection, flushed to API Gateway with `device_id` (A10 §7); API validates farm/trip ownership and commits conditionally to `sack_event_log`. Outbox releases event ONLY after server acknowledges commit.

2. **Scenario 2: Anonymous Public Sack Lookup & Buyer View Projection (Section 8, ASM-058, ASM-094, ASM-096, ASM-154)**:
   - **Public Request**: Buyer scans QR code or enters 80-bit Crockford Base32 sack code `XXXX-XXXX-XXXX-XXXX-C` (ASM-094); sends HTTPS GET request (TLSv1.2_2021) to CloudFront.
   - **WAF Inspection**: AWS WAF enforces rate limit (<= 30 reads per minute per source IP | PD-102); CloudFront forwards to Lambda Function URL via Origin Access Control (OAC).
   - **Architectural Isolation Gate**: Lookup Lambda executes `GetItem` against `buyer_view` DynamoDB table ONLY (CMK `public_view`). Lookup IAM role holds ZERO permissions on `records`, `personal`, or `telemetry` tables and ZERO permissions on their KMS keys.
   - **Data Filtering Invariant**:
     - *4 Groups Returned*: Origin (department, municipality), Processing dates (drying start/end, packing), Status (current sack state), Drying variables (moisture %, grain temperature min/mean/max).
     - *Strictly Withheld*: Farmer name, cédula, phone, subject_id, coordinates, node IDs, raw telemetry, transporter identity, vehicle plate, custody history timestamps, weights, pest records.
   - **Auditing**: Lambda asynchronously writes Event 1 (Sack Read, ASM-069) to `/aws/application/audit` CloudWatch Log Group; subscription filter streams via Data Firehose to Security Account S3 Audit Archive (5-year Object Lock compliance mode).

3. **Scenario 3: Revocation Protocols & Enforced Time Limits (Section 10, ASM-098, PD-100)**:
   - **Overall SLA**: Target to complete revocation within 4 hours of report (ASM-098).
   - **Scenario 3A (Decommissioned Farm Node or Lost Drone Kit)**:
     - Platform Operator authenticates with fresh TOTP (<= 10m) on Operations Screen (or `DeviceRevocation` role fallback | §5.3).
     - Executes 4 atomic IoT Core API actions:
       1. `iot:UpdateCertificate(newStatus='REVOKED')` (certificate preserved revoked for audit trail).
       2. `iot:DetachPolicy(target=certArn)` (revokes publish/connect rights).
       3. `iot:DetachThingPrincipal(principal=certArn)` (disassociates from thing).
       4. `iot:UpdateThing(attributes={status: 'decommissioned'|'lost'})` (silent-node alarm ignores decommissioned node).
     - *Timeline*: Within minutes of leaving `ACTIVE`, IoT Core forcefully terminates active MQTT/TLS sessions and refuses new handshakes (PD-100). Detached policy denies publish. Pre-signed S3 upload URLs for drone expire in 15 minutes (ASM-093). Node local buffer expires and is discarded after 24 hours on device (ASM-074).
   - **Scenario 3B (Departing Drone Pilot or Agronomist)**:
     - Platform Operator executes Cognito IDP revocation actions:
       1. `cognito-idp:AdminDisableUser` (user cannot sign in).
       2. `cognito-idp:AdminUserGlobalSignOut` (revokes active refresh tokens).
       3. `cognito-idp:AdminRemoveUserFromGroup`.
       4. Database sets `revoked_after = NOW()`.
     - *Timeline*: Immediate (0 minutes). API compares token `auth_time` against `revoked_after` on every call; denies any request immediately without waiting for 60-minute token expiry. User account is preserved (disabled) for historical flight auditability.
   - **Scenario 3C (Lost Mobile Phone of Farmer or Transporter)**:
     - Platform Operator executes:
       1. `cognito-idp:AdminUserGlobalSignOut`.
       2. Database marks mobile installation `device_id` as REVOKED (blocks queued outbox uploads).
       3. `cognito-idp:AdminDisableUser` (prevents attacker with stolen SIM from intercepting SMS OTP).
     - *Timeline*: Immediate. Any request with revoked `device_id` is rejected. Outbox events on lost phone never enter platform.
     - *In-Person Re-Enrollment*: User verifies identity in person at Collection Center; operator calls `AdminEnableUser`, updates phone number, and registers new smartphone.

### Source documents

- `docs/security-and-identity.md` (sections 4.3, 5, 8, 10): Authentication flows, sensitive step-up MFA, public buyer view projection and data withholding, and revocation procedures with exact time limits.
- `docs/sack-and-trip.md` (sections 7.2, 10): Public lookup front door (CloudFront, WAF, Lambda URL) and `buyer_view` store.
- `docs/landing-zone.md` (section 7): Audit pipeline for Event 1 (Sack Read) and administrative API auditing.
- `docs/decision-log.md`: ASM-058, ASM-074, ASM-091 to ASM-098, PD-100, PD-102.

### Boxes D01-D13 shown

- `D04`: Public lookup front door (CloudFront, WAF, Lambda Function URL) and API Gateway VPC interface endpoints.
- `D06`: Identities (Cognito User Pools, IAM service roles), KMS customer-managed keys (`public_view`, `records`, `personal`), and access controls.
- `D08`: Farm node device identity and CSR-based certificate lifecycle in AWS IoT Core.
- `D09`: IoT Core topic rule and DynamoDB telemetry store.
