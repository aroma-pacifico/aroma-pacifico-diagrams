# Security and Identity Architecture & Sequence Diagrams

This directory contains the security, identity, encryption, and revocation architecture diagrams and sequence specifications for the Aroma Pacífico coffee traceability platform. All models align with the requirements of the academic DevOps challenge across Cauca, Huila, and Tolima.

## 1. Directory Structure

```text
security/
|-- README.md
|-- identity-and-access.drawio
|-- identity-and-access.svg
|-- identity-and-access.png
|-- revocation-sequence.drawio
|-- revocation-sequence.svg
`-- revocation-sequence.png
```

---

## 2. Identity and Access Architecture (`identity-and-access.drawio`)

### What It Shows

The architecture diagram illustrates the end-to-end identity and access boundary of the Aroma Pacífico platform across its five functional tiers:
1. **Kinds of Identity (Actors)**: Covers workforce administrators, staff application operators, field users, anonymous and registered coffee buyers, edge IoT devices, and automated CI/CD deployment pipelines.
2. **Directories & Registries**: Models identity stores including centralized workforce federation in AWS IAM Identity Center, dual Amazon Cognito user pools (`staff` and `field`), AWS IoT Core device registry for mutual TLS authentication, and GitHub OIDC trust.
3. **Entry Points & Front Doors**: Outlines authenticated perimeters including the AWS Management Console/SSO portal, Amazon API Gateway custom domain with Cognito JWT authorizer, Amazon CloudFront with AWS WAF rate limiting for public sack lookups, and AWS IoT Core MQTT TLS endpoint on port 8883.
4. **Processing & Execution Roles**: Delineates least-privilege Lambda and service roles, dynamic database profile scope evaluation (enforcing immediate 0-minute updates), and strict separation of duties between agricultural engineers (who authorize fumigation) and drone pilots (who execute flights).
5. **KMS CMK Protection & Protected Stores**: Displays the eight customer-managed keys (CMKs) within the workload environment and their protected datastores, as well as cross-account keys for the central audit archive and management budget notifications.

### Source Documents and Sections

- `docs/security-and-identity.md`: Sections 2 (Kinds of Identity), 3 (Application Identity Directory), 4 (Roles and Least Privilege), 5 (MFA and Sign-In), 6 (Device Identities), 7 (Encryption at Rest and in Transit), 8 (Sack Code and Buyer View), 9 (Personal Data and Retention), and 10 (Revocation).
- `docs/landing-zone.md`: Section 4 (Identity and Entry Points) and Section 7 (Audit Architecture).
- `docs/decision-log.md`: ASM-090 through ASM-099, ASM-154, ASM-165, and PD-100.

### Architecture Boxes (D01–D13)

| Box | Component | Implementation in Diagram |
|---|---|---|
| **D06** | Application Identity Directory | Amazon Cognito User Pools (`staff` with mandatory TOTP MFA, and `field` with passwordless one-time codes). |
| **D08** | Centralized Identity Directory | AWS IAM Identity Center in Management Account with SAML 2.0 / IdP federation and permission sets. |
| **D09** | Public Lookup Front Door | Amazon CloudFront distribution and AWS WAF with rate-limiting rules protecting `/s/*`. |
| **D04** | API Gateway / Ingress | Amazon API Gateway REST API with custom domain TLS 1.2+ and Cognito JWT Authorizer. |
| **D02** | IoT Core Broker / Ingress | AWS IoT Core data endpoint on port 8883 enforcing mutual TLS (mTLS) with X.509 client certificates. |

### Identity Kinds and Authentication Matrix

| Identity Kind | Directory | Entry Point | Authentication & MFA | Token Lifetimes |
|---|---|---|---|---|
| **Workforce** | AWS IAM Identity Center (D08) | AWS Console / CLI / SSO Portal | Enterprise IdP (SAML 2.0) + Mandatory MFA | STS session: 1h to 8h |
| **Staff Users** | Cognito User Pool: `staff` (D06) | Amazon API Gateway (D04) | Username + Password + Mandatory TOTP MFA | Access/ID: 30 min, Refresh: 12 h (Step-up: $\le$ 10 min) |
| **Field Users** | Cognito User Pool: `field` (D06) | Amazon API Gateway (D04) | Passwordless SMS OTP (No MFA, justified ASM-091) | Access/ID: 60 min, Refresh: 30 days |
| **Registered Buyers** | Cognito User Pool: `field` (D06) | Amazon API Gateway (D04) | Passwordless Email OTP (No MFA, read-only) | Access/ID: 60 min, Refresh: 1 day |
| **Public Buyers** | None (Unauthenticated) | CloudFront + AWS WAF (D09) | Anonymous HTTPS GET `/s/{sack_code}` | None (WAF rate limit $\le$ 100 req/5 min) |
| **IoT Devices** | AWS IoT Core Registry (D02) | AWS IoT Core MQTT (D02) | X.509 client certificate (mTLS, port 8883) | Permanent cert, instant revocation |
| **CI/CD Pipelines** | GitHub OIDC Provider | AWS STS OIDC Endpoint | Federated OIDC Web Identity Token | Ephemeral STS session: max 1h |

### KMS Customer-Managed Keys (CMKs) Protection

| Account | KMS Key Alias | Protected Store | Scope & Data Classification | Rotation |
|---|---|---|---|---|
| **Workload** | `telemetry` | DynamoDB Telemetry Table & Transporter Position Store | Sensor telemetry (temperature, humidity, solar) and transporter GPS positions | 365 days (ASM-097) |
| **Workload** | `records` | DynamoDB Transactional Tables | Core transactional records: sacks, custody events, trips, orders, grants | 365 days (ASM-097) |
| **Workload** | `personal` | DynamoDB Personal Data Table | Cédula, phone, vehicle plate keyed by `subject_id` (5-year retention, ASM-098) | 365 days (ASM-097) |
| **Workload** | `public_view` | DynamoDB `buyer_view` Projection Table | Isolated public sack projection: origin, dates, status, drying variables | 365 days (ASM-154) |
| **Workload** | `objects` | S3 Drone Object Bucket | Drone photos, orthomosaics, flight polygons, sack history export prefix | 365 days (ASM-097) |
| **Workload** | `restricted` | S3 Drone Object Bucket (`restricted/` prefix) | Drone imagery containing human persons; explicit Deny for all functions | 365 days (ASM-165) |
| **Workload** | `logs` | CloudWatch Logs Groups | Application audit logs, Lambda execution logs, AWSIotLogsV2 | 365 days (ASM-097) |
| **Workload** | `state` | S3 Terraform State Bucket | Remote Terraform state per environment with DynamoDB state locking | 365 days (ADR-003) |
| **Security** | `audit` | S3 Audit Archive Bucket | Organization CloudTrail and application audit records (Object Lock WORM) | 365 days (A08 §7) |
| **Management** | Budget Notification Key | SNS Budget Alerts Topic | Server-side encryption for AWS Budgets notification delivery | 365 days (A08 §6) |

---

## 3. Revocation Sequence Diagram (`revocation-sequence.drawio`)

### What It Shows

The sequence diagram details the three core operational authentication and revocation flows:
1. **Scenario 1 (Authentication Protocols & Sensitive Operation Step-Up)**:
   - *Part 1A*: Staff sign-in flow through Cognito `staff` pool enforcing mandatory TOTP challenge and issuing tokens with 30-minute access/ID and 12-hour refresh validity.
   - *Part 1B*: Sensitive operation step-up authentication ensuring an `auth_time` no older than 10 minutes prior to state-changing operations (such as fumigation authorization, threshold configuration, or identity administration), and enforcing separation of duties between agronomist and pilot.
   - *Part 1C*: Field user sign-in via passwordless SMS OTP with compensating controls (hardware keystore, phone lock, and dynamic database scopes) and offline outbox queuing.
2. **Scenario 2 (Anonymous Public Sack Lookup & Buyer View)**:
   - Validates public sack inquiries via 80-bit Crockford Base32 sack codes and check symbols.
   - Demonstrates perimeter protection through AWS WAF rate limiting ($\le$ 100 requests per 5 minutes per source IP on `/s/*`).
   - Enforces complete data isolation: lookup Lambda queries only the `buyer_view` projection table encrypted with CMK `public_view`, holding zero access to internal transactional or personal tables.
   - Enforces uniform responses with strict security headers: `Cache-Control: no-store`, `X-Robots-Tag: noindex`, and `Referrer-Policy: no-referrer`.
3. **Scenario 3 (Revocation Protocols & Enforced Time Limits)**:
   - *Part 3A*: Decommissioning a farm sensor node or lost drone kit via `UpdateCertificate(REVOKED)`, terminating broker MQTT connections in minutes (PD-100) and expiring upload grants within 15 minutes (ASM-093).
   - *Part 3B*: Departing staff member invalidation via `AdminDisableUser` and `AdminUserGlobalSignOut` with immediate token denial through database `revoked_after` timestamps.
   - *Part 3C*: Lost mobile device handling for field users, including session invalidation, device denylisting, and in-person cédula verification for phone re-enrollment.

### Source Documents and Sections

- `docs/security-and-identity.md`: Section 5 (MFA and Sign-In), Section 8 (Sack Code and Buyer View), and Section 10 (Revocation).
- `docs/sack-and-trip.md`: Section 7.2 (Public Front Door), Section 7.4 (Caching), and Section 10 (Stores).
- `docs/decision-log.md`: ASM-058, ASM-091, ASM-092, ASM-093, ASM-094, ASM-095, ASM-098, ASM-154, ASM-155, and PD-100.

### Enforced Time Limits and SLA Protocols

| Event / Mechanism | Enforced Time Limit | Normative Reference | Operational Control |
|---|---|---|---|
| **Incident Reporting Duty** | $\le$ 2 hours from awareness | ASM-098 | Mandatory notification by contract company, Federation liaison, or collection center. |
| **Cloud Revocation SLA** | $\le$ 4 hours target from report | ASM-098 | Target completion SLA for platform operators to execute revocation across directories. |
| **IoT Broker Disconnection** | Minutes ($\approx$ 0 min) | PD-100 | `UpdateCertificate(REVOKED)` causes immediate mTLS handshake failure and drops active sessions. |
| **S3 Pre-Signed Upload Grants** | $\le$ 15 minutes max | ASM-093 | Drone kit upload grants expire automatically; invalidation prevents subsequent uploads. |
| **Step-Up Authentication** | $\le$ 10 minutes max | ASM-092 | Operations requiring step-up reject tokens where `now - auth_time > 10 min`. |
| **Staff Token Lifetimes** | Access/ID: 30m, Refresh: 12h | ASM-092 | Enforced by Cognito `staff` user pool settings. |
| **Field Token Lifetimes** | Access/ID: 60m, Refresh: 30d | ASM-092 | Enforced by Cognito `field` user pool settings (Buyer refresh: 1 day). |
| **Token Invalidation** | Immediate (0 min) | ASM-098 | `AdminUserGlobalSignOut` revokes refresh tokens; `revoked_after` checks reject active access tokens. |
| **Public Lookup Rate Limit** | $\le$ 100 req / 5 min per IP | ASM-155 | AWS WAF rate-based rule blocks abusive source IPs with HTTP 429. |
| **Personal Data Retention** | 5 years | ASM-098 | Personal data table TTL clock keyed by pseudonymous `subject_id`. |

---

## 4. Architectural Decisions and Traceability

All values, configurations, and boundaries depicted in these diagrams are grounded directly in the referenced architecture specifications and registered decision-log entries:
- **ASM-090**: Amazon Cognito user pools on the Essentials plan in `us-east-1` with dual `staff` and `field` pools per environment in the workload account.
- **ASM-091**: Justified absence of MFA for farmers and transporters; passwordless SMS one-time codes with keystore encryption and server-side dynamic scoping.
- **ASM-092**: Token lifetimes (staff: 30m access/ID, 12h refresh; field: 60m access/ID, 30d refresh; buyer refresh: 1d) and 10-minute step-up requirement for sensitive staff operations.
- **ASM-093**: Drone kit paired ground station identity in AWS IoT Core; kit upload grants with 15-minute pre-signed URLs.
- **ASM-094**: 80-bit random Crockford Base32 sack code with check symbol printed with public lookup QR code.
- **ASM-095**: Edge outbox encryption with phone hardware keystore; contracted drone operator encrypts ground station storage.
- **ASM-096**: Buyer view granularity restricted strictly to origin, processing dates, status, and drying variables.
- **ASM-097**: One customer-managed key per data class and environment, with automatic 365-day rotation.
- **ASM-098**: Retention clock of 5 years for personal data, 2-hour reporting duty, and 4-hour target cloud revocation SLA.
- **ASM-099**: Single role per identity; threshold configuration and enrollment restricted to platform operators.
- **ASM-154**: Isolated `public_view` KMS key encrypting the `buyer_view` projection table.
- **ASM-155**: Public lookup AWS WAF rate limit ($\le$ 100 req/5 min per source IP) and reserved concurrency of 10.
- **ASM-165**: Restricted drone object prefix (`restricted/`) with isolated CMK `restricted` denying all function execution roles.
- **PD-100**: Live verification of AWS IoT Core revocation mechanics and MQTT session termination.
