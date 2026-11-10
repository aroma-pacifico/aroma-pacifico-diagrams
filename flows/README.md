# Flows Diagrams

This directory contains the canonical architectural state machine and end-to-end operational sequence diagrams for the Aroma Pacífico coffee traceability platform.

---

## Sack State Machine and Lifecycle

- Source diagram: [`sack-states.drawio`](sack-states.drawio)
- Vector export: [`sack-states.svg`](sack-states.svg)
- Bitmap preview: [`sack-states.png`](sack-states.png)

### What it shows

This diagram models the complete lifecycle of a coffee sack across the seven states specified in Part C of the case statement and Design System badges (`in_preparation`, `in_transit_to_center`, `at_center_accepted`, `at_center_rejected`, `in_transit_to_port`, `delivered_at_port_accepted`, and `delivered_at_port_rejected`). It captures operational transitions T1 through T8 with their triggering events, responsible actors (Farmer, Transporter, and Collection-Center Operator), and guard conditions, alongside the offline receiver-authoritative reconciliation rule (ASM-152) and the five non-state-changing events (ASM-151). The lower panels detail the append-only DynamoDB event store topology under customer-managed KMS keys (`records`, `telemetry`, `public_view`), the deterministic history replay fold, and disaster recovery objectives (RPO 15 minutes, RTO 4 hours).

### Source documents

| Source Document | Sections | Covered Topics and Decisions |
|---|---|---|
| `aroma-pacifico-architecture/docs/sack-and-trip.md` | Sec 1–4, 6, 8–10 | Sack lifecycle, transition table T1–T8, offline fold rules, event schema, center reserve (>= 400 codes), store topology, RPO 15 min / RTO 4 h |
| `aroma-pacifico-architecture/docs/security-and-identity.md` | Sec 3.2, 4.1–4.2, 7.2, 8 | Cognito `staff` (MFA) and `field` user pools, KMS keys (`records`, `telemetry`, `public_view`), role catalog |
| `aroma-pacifico-architecture/docs/architecture.md` | Sec 14 | Box catalog D01–D13 and IaC module traceability |
| `aroma-pacifico-architecture/docs/decision-log.md` | Decision entries | ASM-007, ASM-057, ASM-076, ASM-077, ASM-079, ASM-091, ASM-094, ASM-096, ASM-150–159, PD-096, PD-110, PD-151, PD-153, PD-154 |

### Boxes D01–D13 shown

| Box | Name | Architectural Role in Diagram |
|---|---|---|
| `D04` | Application Services | VPC private application tier hosting Lambda handlers (`sack-events`, `sack-applier`, `sack-exporter`) and API endpoints |
| `D05` | Collection Center Operations | Receiving post in Cauca, Huila, and Tolima collection centers, managing local physical intake and outbound staging |
| `D06` | Data Stores & Security | DynamoDB tables (`sack_event_log`, `sacks`, `sack_history`, `trips`, `buyer_view`), Cognito pools, and customer KMS keys |

---

## Sack, Trip and Buyer Public Lookup Sequence

- Source diagram: [`sack-and-trip-sequence.drawio`](sack-and-trip-sequence.drawio)
- Vector export: [`sack-and-trip-sequence.svg`](sack-and-trip-sequence.svg)
- Bitmap preview: [`sack-and-trip-sequence.png`](sack-and-trip-sequence.png)

### What it shows

This diagram illustrates the end-to-end operational sequence of sack registration, transport custody, collection center reception, and unauthenticated public query across six distinct phases. It details pre-printed code sheet generation at the collection center via IPP over TCP 631, wet-mill sack binding and scale weighing with offline outbox buffering on the Farmer App, 5-minute GNSS tracking during transporter transit across cellular dead zones, center reception with reserve label binding for unlabeled arrivals, and highway transit to Buenaventura port. Phase 6 models the unauthenticated public lookup architecture via Amazon CloudFront, AWS WAF rate-limiting, and Lookup Lambda URL with Origin Access Control (OAC), demonstrating strict isolation of the `buyer_view` DynamoDB projection, anonymous audit logging to `/aws/application/audit`, and the security boundary withholding producer personal data and internal telemetry.

### Source documents

| Source Document | Sections | Covered Topics and Decisions |
|---|---|---|
| `aroma-pacifico-architecture/docs/sack-and-trip.md` | Sec 2–7, 8–10 | Registration at farm, custody handover, 5-min GNSS fixes, code sheets, public lookup, peak load arithmetic, store schema |
| `aroma-pacifico-architecture/docs/security-and-identity.md` | Sec 3.2, 4.2, 7.2, 8.1–8.4, 9.1 | Cognito pools, KMS key separation, Crockford Base32 format, buyer view projection, withheld sensitive fields (cédula, GPS) |
| `aroma-pacifico-architecture/docs/connectivity.md` | Sec 5, 7, 10 | Store-and-forward outbox buffering, priority drain order (custody first, positions oldest first) |
| `aroma-pacifico-architecture/docs/firewall-policy.md` | Rules CL-PUB-01 to CL-PUB-04 | Edge WAF Web ACL inspection, rate limits (100 req / 5 min per IP), CloudFront OAC |
| `aroma-pacifico-architecture/docs/landing-zone.md` | Sec 7.2 | Anonymous lookup audit in `/aws/application/audit` CloudWatch log group |
| `aroma-pacifico-architecture/docs/architecture.md` | Sec 14 | Box catalog D01–D13 and IaC module traceability |
| `aroma-pacifico-architecture/docs/decision-log.md` | Decision entries | ASM-057, ASM-058, ASM-069, ASM-076, ASM-077, ASM-079, ASM-094, ASM-096, ASM-150–159, PD-051, PD-096, PD-102, PD-110, PD-151, PD-153, PD-154 |

### Boxes D01–D13 shown

| Box | Name | Architectural Role in Diagram |
|---|---|---|
| `D04` | Application Services | VPC private application tier and serverless compute (Lambda handlers, Lookup Function URL with OAC) |
| `D05` | Collection Center Operations | Center receiving post and local network IPP code-sheet printer (TCP 631) |
| `D06` | Data Stores & Security | DynamoDB tables (`sack_event_log`, `sacks`, `trips`, `transport_positions`, `buyer_view`), CloudWatch Logs audit, customer KMS keys |
| `D07` | Public Lookup Edge Security | Amazon CloudFront distribution and AWS WAF web ACL (`CL-PUB-01`) enforcing TLS 1.2+ termination and rate limiting |
