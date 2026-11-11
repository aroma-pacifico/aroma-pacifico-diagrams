# Security, Identity, and Firewall Architecture & Sequence Diagrams

This directory contains the security, identity, encryption, perimeter and cloud firewall architecture diagrams and sequence specifications for the Aroma Pacífico coffee traceability platform. All models align with the requirements of the academic DevOps challenge across Cauca, Huila, and Tolima.

## 1. Directory Structure

```text
security/
|-- README.md
|-- firewall-policy.drawio
|-- firewall-policy.svg
|-- firewall-policy.png
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
2. **Directories & Registries**: Models identity stores including centralized workforce directory in AWS IAM Identity Center (D06), dual Amazon Cognito user pools (`staff` and `field`, D06), AWS IoT Core device registry for farm node certificates (D08), and GitHub OIDC trust.
3. **Entry Points & Front Doors**: Outlines unauthenticated and authenticated perimeters including the AWS Management Console/SSO portal, Amazon API Gateway custom domain with Cognito JWT authorizer, Amazon CloudFront with AWS WAF rate limiting for public sack lookups, and AWS IoT Core MQTT TLS endpoint on port 8883.
4. **Processing & Execution Roles**: Delineates least-privilege Lambda and service roles, dynamic database profile scope evaluation (enforcing immediate profile scoping), telemetry ingestion in the IoT rules engine (D09), and strict separation of duties between agricultural engineers (who authorize fumigation) and drone pilots (who execute flights).
5. **KMS CMK Protection & Protected Stores**: Displays the eight customer-managed keys (CMKs) within the workload environment and their protected datastores (including the S3 state bucket and `state` key badged D02), as well as cross-account keys for the central audit archive and management budget notifications.

### Source Documents and Sections

- `docs/security-and-identity.md`: Sections 2 (Kinds of Identity), 3 (Application Identity Directory), 4 (Roles and Least Privilege), 5 (MFA and Sign-In), 6 (Device Identities), 7 (Encryption at Rest and in Transit), 8 (Sack Code and Buyer View), 9 (Personal Data and Retention), and 10 (Revocation).
- `docs/landing-zone.md`: Section 4 (Identity and Entry Points) and Section 7 (Audit Architecture).
- `docs/architecture.md`: Section 14 (Diagram boxes D01 to D13 and traceability to IaC).
- `docs/decision-log.md`: ASM-090 through ASM-099, ASM-154, ASM-165, and PD-100.

### Architecture Boxes (D01–D13 Traceability)

In accordance with `docs/architecture.md` section 14, components are badged only with the box that creates or implements them:

| Box | Component / Implementation | Track B Issue & Scope |
|---|---|---|
| **D06** | Identities, KMS keys and initial access controls | B06 (`#26`): AWS IAM Identity Center, Cognito User Pools (`staff` and `field`), and workload KMS CMKs (`telemetry`, `records`, `personal`, `public_view`, `objects`, `restricted`, `logs`, `state`). |
| **D02** | One S3 state bucket per environment with native locking | B02 (`#22`): S3 Terraform State Bucket and `state` KMS CMK key. |
| **D08** | The reusable farm module: one pattern farm | B08 (`#28`): Farm node IoT device identity in AWS IoT Core Registry and X.509 client certificate from CSR. |
| **D09** | Telemetry ingestion pattern module | B09 (`#29`): AWS IoT Core MQTT rules engine topic routing and ingestion into DynamoDB. |

*Note: Infrastructure entry points such as Amazon API Gateway (part of workload networking) and Amazon CloudFront with AWS WAF (public lookup front door) do not map to separate B-track boxes and remain unbadged.*

### Identity Kinds and Authentication Matrix

| Identity Kind | Directory | Entry Point | Authentication & MFA | Session / Token Lifetimes |
|---|---|---|---|---|
| **Workforce** | AWS IAM Identity Center (D06) | AWS Console / CLI / SSO Portal | Mandatory MFA on every sign-in (§5.3) | `PlatformReadOnly`/`Auditor`: 4h; `DeviceRevocation`/`IdentitySupport`/`BreakGlass`: 1h (§5.3) |
| **Staff Users** | Cognito User Pool: `staff` (D06) | Amazon API Gateway | Username + Password + Mandatory TOTP MFA | Access/ID: 30 min, Refresh: 12 h (Step-up: $\le$ 10 min) |
| **Field Users** | Cognito User Pool: `field` (D06) | Amazon API Gateway | Passwordless SMS OTP (No MFA, justified ASM-091) | Access/ID: 60 min, Refresh: 30 days |
| **Registered Buyers** | Cognito User Pool: `field` (D06) | Amazon API Gateway | Passwordless Email OTP (No MFA, read-only) | Access/ID: 60 min, Refresh: 1 day |
| **Public Buyers** | None (Unauthenticated) | CloudFront + AWS WAF | Anonymous HTTPS GET `/s/{sack_code}` | None (WAF rate limit $\le$ 100 req/5 min) |
| **IoT Devices** | AWS IoT Core Registry (D08) | AWS IoT Core MQTT | X.509 client certificate (mTLS, port 8883) | Revocation takes effect within minutes of leaving ACTIVE |
| **CI/CD Pipelines** | GitHub OIDC Provider | AWS STS OIDC Endpoint | Federated OIDC Web Identity Token | Short-lived role credentials (§3.4, ADR-003) |

### KMS Customer-Managed Keys (CMKs) Protection

| Account | KMS Key Alias | Protected Store | Scope & Data Classification | Rotation |
|---|---|---|---|---|
| **Workload** | `telemetry` (D06) | DynamoDB Telemetry Table & Position Store | Sensor telemetry (temperature, humidity, solar) and transporter GPS positions | 365 days (ASM-097) |
| **Workload** | `records` (D06) | DynamoDB Transactional Tables | Core transactional records: sacks, custody events, trips, orders, grants | 365 days (ASM-097) |
| **Workload** | `personal` (D06) | DynamoDB Personal Data Table | Cédula, phone, vehicle plate keyed by `subject_id` (5-year retention, ASM-098) | 365 days (ASM-097) |
| **Workload** | `public_view` (D06) | DynamoDB `buyer_view` Projection Table | Isolated public sack projection: origin, dates, status, drying variables | 365 days (ASM-154) |
| **Workload** | `objects` (D06) | S3 Drone Object Bucket | Drone photos, orthomosaics, flight polygons, sack history export prefix | 365 days (ASM-097) |
| **Workload** | `restricted` (D06) | S3 Drone Object Bucket (`restricted/` prefix) | Drone imagery containing human persons; explicit Deny for all functions | 365 days (ASM-165) |
| **Workload** | `logs` (D06) | CloudWatch Logs Groups | Application audit logs, Lambda execution logs, AWSIotLogsV2 | 365 days (ASM-097) |
| **Workload** | `state` (D02 / D06) | S3 Terraform State Bucket (D02) | Remote Terraform state per environment with DynamoDB state locking | 365 days (ADR-003, B02) |
| **Security** | `audit` | S3 Audit Archive Bucket | Organization CloudTrail and application audit records (Object Lock WORM) | 365 days (A08 §7) |
| **Management** | Budget Notification Key | SNS Budget Alerts Topic | Server-side encryption for AWS Budgets notification delivery | 365 days (A08 §6) |

---

## 3. Revocation Sequence Diagram (`revocation-sequence.drawio`)

### What It Shows

The sequence diagram details the three core operational authentication and revocation flows:
1. **Scenario 1 (Authentication Protocols & Sensitive Operation Step-Up)**:
   - *Part 1A*: Staff sign-in flow through Cognito `staff` pool (D06) enforcing mandatory TOTP challenge and issuing tokens with 30-minute access/ID and 12-hour refresh validity.
   - *Part 1B*: Sensitive operation step-up authentication ensuring an `auth_time` no older than 10 minutes prior to state-changing operations (such as fumigation authorization, threshold configuration, or identity administration), and enforcing separation of duties between agronomist and pilot.
   - *Part 1C*: Field user sign-in via passwordless SMS OTP with compensating controls (hardware keystore, phone lock, and dynamic database scopes) and offline outbox queuing.
2. **Scenario 2 (Anonymous Public Sack Lookup & Buyer View)**:
   - Validates public sack inquiries via 80-bit Crockford Base32 sack codes and check symbols.
   - Demonstrates perimeter protection through AWS WAF rate limiting ($\le$ 100 requests per 5 minutes per source IP on `/s/*`).
   - Enforces complete data isolation: lookup Lambda queries only the `buyer_view` projection table encrypted with CMK `public_view`, holding zero access to internal transactional or personal tables.
   - Enforces uniform responses with strict security headers: `Cache-Control: no-store`, `X-Robots-Tag: noindex`, and `Referrer-Policy: no-referrer`.
3. **Scenario 3 (Revocation Protocols & Enforced Time Limits)**:
   - *Part 3A*: Decommissioning a farm sensor node or lost drone kit via `UpdateCertificate(REVOKED)`, terminating broker MQTT connections within minutes of leaving ACTIVE (PD-100) and expiring upload grants within 15 minutes (ASM-093).
   - *Part 3B*: Departing staff member invalidation via `AdminDisableUser` and `AdminUserGlobalSignOut` with immediate token denial through database `revoked_after` timestamps.
   - *Part 3C*: Lost mobile device handling for field users, including session invalidation, device denylisting, and in-person cédula verification for phone re-enrollment.

### Source Documents and Sections

- `docs/security-and-identity.md`: Section 5 (MFA and Sign-In), Section 8 (Sack Code and Buyer View), and Section 10 (Revocation).
- `docs/sack-and-trip.md`: Section 7.2 (Public Front Door), Section 7.4 (Caching), and Section 10 (Stores).
- `docs/architecture.md`: Section 14 (Box mapping).
- `docs/decision-log.md`: ASM-058, ASM-091, ASM-092, ASM-093, ASM-094, ASM-095, ASM-098, ASM-154, ASM-155, and PD-100.

### Enforced Time Limits and SLA Protocols

| Event / Mechanism | Enforced Time Limit | Normative Reference | Operational Control |
|---|---|---|---|
| **Incident Reporting Duty** | As soon as known | ASM-098 | Mandatory notification by contract company, Federation liaison, or collection center. |
| **Cloud Revocation SLA** | $\le$ 4 hours target from report | ASM-098 | Target completion SLA for platform operators to execute revocation across directories. |
| **IoT Broker Disconnection** | Within minutes of leaving ACTIVE | PD-100, §10 | `UpdateCertificate(REVOKED)` terminates MQTT session and prevents reconnection. |
| **S3 Pre-Signed Upload Grants** | $\le$ 15 minutes max | ASM-093 | Drone kit upload grants expire automatically; invalidation prevents subsequent uploads. |
| **Step-Up Authentication** | $\le$ 10 minutes max | ASM-092 | Operations requiring step-up reject tokens where `now - auth_time > 10 min`. |
| **Staff Token Lifetimes** | Access/ID: 30m, Refresh: 12h | ASM-092 | Enforced by Cognito `staff` user pool settings. |
| **Field Token Lifetimes** | Access/ID: 60m, Refresh: 30d | ASM-092 | Enforced by Cognito `field` user pool settings (Buyer refresh: 1 day). |
| **Token Invalidation** | Immediate | docs/security-and-identity.md §10 | `AdminUserGlobalSignOut` revokes refresh tokens; active access tokens refused at once through database `revoked_after` check (§10). |
| **Public Lookup Rate Limit** | $\le$ 100 req / 5 min per IP | ASM-155 | AWS WAF rate-based rule blocks requests from source IPs exceeding rate budget (ASM-155, sack-and-trip.md §7.3). |
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
- **ASM-098**: Retention clock of 5 years for personal data, reporting duty as soon as known, and 4-hour target cloud revocation SLA.
- **ASM-099**: Single role per identity; threshold configuration and enrollment restricted to platform operators.
- **ASM-154**: Isolated `public_view` KMS key encrypting the `buyer_view` projection table.
- **ASM-155**: Public lookup AWS WAF rate limit ($\le$ 100 req/5 min per source IP) and reserved concurrency of 10.
- **ASM-165**: Restricted drone object prefix (`restricted/`) with isolated CMK `restricted` denying all function execution roles.
- **PD-100**: Live verification of AWS IoT Core revocation mechanics and MQTT session termination within minutes of leaving ACTIVE.

## 4. Perimeter and Cloud Firewall Policy Architecture (`firewall-policy.drawio`)

### What It Shows

The architecture diagram illustrates the end-to-end perimeter and cloud firewall policy for the Aroma Pacífico coffee traceability pilot across the collection centers and AWS workload environment:
1. **Collection Center Perimeter (`10.4x.0.0/16`)**: Covers the three collection centers (Cauca `10.41.0.0/16`, Huila `10.42.0.0/16`, Tolima `10.43.0.0/16`) and their segmented LAN zones:
   - *Gerencia (`10.4x.1.0/24`, VLAN 101, WPA3-Enterprise)*: Office management workstations with 802.1X EAP-TLS.
   - *Soporte (`10.4x.2.0/24`, VLAN 102, WPA3-Enterprise)*: Technical staff and the paired drone ground station (`10.4x.2.10/32`).
   - *Logística (`10.4x.3.0/24`, VLAN 103, WPA3-Enterprise)*: Coffee intake workstations, barcode scanners, and code-sheet printers.
   - *Servidores locales (`10.4x.10.0/24`, VLAN 110, Wired)*: Core switches, access points, network code-sheet printers (`.50` to `.59`), and temporary commissioning laptop (`.60`).
   - *Internet-Only Isolated Context (VRF)*: Separate guest and transporter wireless networks (*Transporte `10.4x.4.0/24`*, *Visitantes `10.4x.5.0/24`*) strictly denied routing to logistics, management, or internal networks.
   - *Perimeter Firewall (`10.4x.20.1`)*: Enforces local gateway services (DHCP, DNS, NTP), priority queue bandwidth shaping (2 Mbps network, 15 Mbps logistics, 20 Mbps drone, 25 Mbps staff, 8 Mbps guest), and WAN interfaces.
2. **Public Internet & Central Transit Hub**:
   - *Public Clients*: Farm field nodes (telemetry via MQTT/TLS), transporter and farmer mobile apps (Cognito-authenticated REST APIs), public sack lookup consumers (CloudFront/WAF rate-limited 30/min), and municipal drone operators.
   - *AWS Network Account Central Transit Hub (`tgw-aroma-hub`)*: Route tables for `centers`, `pdn`, `dev`, and `qa`; AWS Site-to-Site VPN attachments (IKEv2, UDP 500/4500, IP 50 ESP); customer-managed prefix lists shared via AWS RAM (`cc-firewalls`, `cc-staff`, `cc-logistics`, `cc-ground-stations`, `cc-device-mgmt`).
   - *Isolation Guarantees*: Blackhole routes preventing center-to-center lateral traversal (`10.40.0.0/13`) and isolating development environments (`10.80.0.0/16`).
3. **AWS Workload Account Production (`pdn` `10.82.0.0/16`) Environment**:
   - *Regional Managed Public Entry Points*: AWS IoT Core data endpoint (`CL-TEL-01`, `CL-TEL-02`), public application API front door (`CL-TEL-03`), public sack lookup front door (`CL-PUB-01`, `CL-PUB-02`), and Amazon S3 Regional Endpoint (`CL-DRN-03`).
   - *Production VPC Tiers (3 AZs)*: `transit` (/28 subnets), `application` (`10.82.48.0/20`), `object_storage` (`10.82.80.0/20`), `administration` (`10.82.96.0/20`), `data` (`10.82.64.0/20`), and `public_query` (`10.82.0.0/20`).
   - *Explicit Reserved Tiers*: `drone_object_ingress` (`10.82.32.0/20`, ASM-140 closing PD-121) and `telemetry_ingress` (`10.82.16.0/20`, A10 section 4.2), verifying zero active endpoints and explicit denies from on-premise networks.
   - *Cloud Defense Controls*: Egress isolation (no NAT Gateway, no `0.0.0.0/0` internet route, `CL-EGR-01`), explicit default deny (`CL-DEF-01`), VPC Flow Logs capture to CloudWatch Logs with KMS encryption (`CL-DEF-02`), and AWS Config rules blocking open management ports (`CL-ADM-05`).
   - *Non-Production & Prepared Environments*: QA environment (`10.81.0.0/16`, temporary test path `CC-LNK-06` / `CL-LNK-05`), Dev environment (`10.80.0.0/16`, isolated), and Azure prepared target (`10.88.0.0/13`, ADR-001).

### Source Documents and Sections

- `docs/firewall-policy.md`:
  - Section 2: Rule identifier format (`CC-*` and `CL-*`).
  - Section 3: Rule groups (`GST`, `BAS`, `LNK`, `ADM`, `DRN`, `LOG`, `STF`, `TEL`, `PUB`, `APP`, `DAT`, `EGR`, `DEF`).
  - Section 4: Collection center perimeter policy (10-rule group top-down matrix).
  - Section 5: Managed rule groups, customer-managed prefix lists, and AWS RAM sharing.
  - Section 6: Non-production environments, temporary rules, and audit schedules.
  - Section 7: Audit rules before production (`PD-125`).
  - Section 8: Traffic control, priority classes, and bandwidth guarantees.
- `docs/collection-center-network.md`:
  - Section 6: Perimeter firewall rules and interface mappings.
  - Section 7: Device management IP allocation (`10.4x.10.0/24`).
  - Section 8: Bandwidth allocation, priority queues, and traffic shaping.
- `docs/architecture.md`:
  - Section 12: Network topology and IP allocations.
  - Section 14: Architecture box catalog (D04, D05, D07).
- `docs/decision-log.md`: PD-011, PD-052, PD-083, PD-091, PD-093, PD-100, PD-102, PD-120 through PD-125, ASM-078, ASM-120 through ASM-129, and ASM-140.

### Architecture Boxes (D01–D13 Traceability)

In accordance with `docs/architecture.md` section 14:

| Box | Component / Implementation | Track B Issue & Scope |
|---|---|---|
| **D04** | VPC and tiers of pattern environment (`10.82.0.0/16`) | B04 (`#24`): Covers transit, application, object_storage, administration, data, and public_query tiers, along with reserved tier boundaries. |
| **D05** | Association of environment with one pattern collection center (`10.4x.0.0/16`) | B05 (`#25`): Site-to-Site VPN connections from Cauca (.41), Huila (.42), and Tolima (.43) to `tgw-aroma-hub` and TGW route table associations. |
| **D07** | Pattern firewall rules (security module, Epic 6) | B07 (`#27`): Comprehensive 13-rule-group matrix across LAN perimeter interfaces, Transit Gateway, Security Groups, Stateless NACLs, and AWS WAF Web ACLs. |

### Visual Dash Style Legend

- **Solid Blue (`#2563eb`)**: Permanent Rule (Subject to scheduled review).
- **Dashed Blue (`#2563eb`)**: Audit before Production (`PD-125`).
- **Dotted Amber (`#d97706`)**: Temporary Rule (Active during acceptance/maintenance window; flagged for audit).
- **Dashed Red (`#dc2626`)**: Explicit Deny / Blackhole Route.

---

## 5. Architectural Decisions and Traceability

All values, configurations, and boundaries depicted in these diagrams are grounded directly in the referenced architecture specifications and registered decision-log entries:
- **ASM-090**: Amazon Cognito user pools on the Essentials plan in `us-east-1` with dual `staff` and `field` pools per environment in the workload account.
- **ASM-091**: Justified absence of MFA for farmers and transporters; passwordless SMS one-time codes with keystore encryption and server-side dynamic scoping.
- **ASM-092**: Token lifetimes (staff: 30m access/ID, 12h refresh; field: 60m access/ID, 30d refresh; buyer refresh: 1d) and 10-minute step-up requirement for sensitive staff operations.
- **ASM-093**: Drone kit paired ground station identity in AWS IoT Core; kit upload grants with 15-minute pre-signed URLs.
- **ASM-094**: 80-bit random Crockford Base32 sack code with check symbol printed with public lookup QR code.
- **ASM-095**: Edge outbox encryption with phone hardware keystore; contracted drone operator encrypts ground station storage.
- **ASM-096**: Buyer view granularity restricted strictly to origin, processing dates, status, and drying variables.
- **ASM-097**: One customer-managed key per data class and environment, with automatic 365-day rotation.
- **ASM-098**: Retention clock of 5 years for personal data, reporting duty as soon as known, and 4-hour target cloud revocation SLA.
- **ASM-099**: Single role per identity; threshold configuration and enrollment restricted to platform operators.
- **ASM-154**: Isolated `public_view` KMS key encrypting the `buyer_view` projection table.
- **ASM-155**: Public lookup AWS WAF rate limit ($\le$ 100 req/5 min per source IP) and reserved concurrency of 10.
- **ASM-165**: Restricted drone object prefix (`restricted/`) with isolated CMK `restricted` denying all function execution roles.
- **PD-100**: Live verification of AWS IoT Core revocation mechanics and MQTT session termination within minutes of leaving ACTIVE.
- **ASM-078**: IPsec VPN tunnel establishment (IKEv2, UDP 500/4500, IP 50 ESP) with automatic cellular failover.
- **ASM-120**: Primary and secondary Site-to-Site VPN tunnels terminating at AWS Transit Gateway in us-east-1.
- **ASM-121**: Scoped temporary routing between collection center logistics VLAN and QA application tier for acceptance testing.
- **ASM-122**: Conditional DNS forwarding from collection center firewalls to Route 53 Resolver inbound endpoints in administration tier.
- **ASM-123**: Local RADIUS authentication on perimeter firewall validated against centralized certificate authority.
- **ASM-124**: Bandwidth guarantees and shaping across center WAN interfaces (100 Mbps primary, LTE backup).
- **ASM-125**: Zero direct database access from collection centers or external networks; data tier accessible exclusively from application tier.
- **ASM-126**: Blackhole routing on Transit Gateway preventing lateral traversal between collection centers.
- **ASM-127**: Cloud workload private tier egress isolation with zero NAT Gateways and no default internet route.
- **ASM-128**: Customer-managed prefix lists shared via AWS RAM for centralized security group and route management.
- **ASM-129**: AWS WAF rate-limiting rules and AWSManagedRules rule groups protecting public entry points.
- **ASM-140**: Explicit reservation of drone_object_ingress tier (10.82.32.0/20) with zero active endpoints in pilot (closing PD-121).
- **PD-102**: Public lookup rate limiting budget calibrated to 30 requests/minute per source IP.
- **PD-125**: Pre-production audit mandate for firewall rules marked for verification before production traffic enablement.

