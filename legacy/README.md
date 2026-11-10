# Legacy Diagrams

This folder contains the architectural legacy traceability diagrams for the Aroma Pacífico coffee traceability platform.

## 2020 Legacy Mapping: Keep, Retire and Replace

`legacy-2020.drawio` is the editable, uncompressed UTF-8 Draw.io XML source. `legacy-2020.svg` is the vector publication export with embedded diagram data, and `legacy-2020.png` is the 2400-pixel high-resolution preview export.

### What the diagram shows

The diagram maps every element of the 2020 pilot design identified in the case statement (`docs/legacy-2020.md`) to its authoritative 2026 architectural outcome: Keep, Retire or Replace.

| ID | 2020 Element | Decision | 2026 Target Design | Source Section | IaC Box |
|---|---|---|---|---|---|
| L-01 | Traceability from farm to buyer, through wet mill, transport, collection center and port | Keep | End-to-end 60 kg sack traceability with 5-year compliance audit log (S3 Object Lock, ASM-067) | A16 (Sack &amp; Trip §3) | None |
| L-02 | Geographic scope: Cauca, Huila and Tolima | Keep | Three regional collection centers allocated RFC 1918 supernet 10.40.0.0/13 (Cauca 10.41.0.0/16, Huila 10.42.0.0/16, Tolima 10.43.0.0/16; 4th department reserved at 10.44.0.0/16, ASM-030) | A07 (Addressing §2) | D05 |
| L-03 | Crop and drying monitoring with an alert to the farmer | Keep | 2 nodes/farm (crop 6/day, drying 24/day). IoT Core mTLS ingestion, DynamoDB telemetry store, deduplicated alert episodes (ASM-135) via SMS and mobile push | A10 (Farm Monitoring §1) | D08, D09, D10 |
| L-04 | Harvest record and packing in 60 kg sacks | Keep | Scale-measured digital weight recorded at wet mill registration with 60 kg nominal commercial reference (ASM-150) and cryptographic sack ID generation | A16 (Sack &amp; Trip §2) | None |
| L-05 | Journey tracking to the collection center and to Buenaventura | Keep | Two-leg overland journey tracking (farm-to-center and center-to-port) via transporter app with 5-min GNSS fixes (transport_position) and custody handoffs (sack_custody) | A10 / A16 (Sack &amp; Trip §5) | None |
| L-06 | Lookup of a sack by code for the farmer, the collection center and the buyer | Keep | Unauthenticated public sack lookup front door via CloudFront + AWS WAF + Lambda URL (ASM-058, PD-051), restricted to buyer view (cédula and internal trace withheld) | A09 / A16 (Sack &amp; Trip §7) | D04 |
| L-07 | Drone service: delimit the lot, confirm the alert, fumigate, verify and estimate the harvest | Keep | All five flight missions preserved under combined flight mode (contracted operator, ASM-141) with agronomist MFA authorization and S3 flight record audit | ADR-002 / A15 (Drone Service §5.2) | None |
| L-08 | Three applications: farmer, transporter, and web reports with sack code reading | Keep | Target application API contracts (REST/GraphQL), Cognito User Pools (staff MFA / field passwordless), and ingestion backends delivered for planned client applications | A09 (Automation &amp; Pipelines §2) | None |
| L-09 | Assumption of data coverage on most farms | Replace | Explicit offline behavior for every data flow; satellite IP bearer for farms out of cellular coverage (dual-mode modules, contact at least every 4h, ASM-073, PD-081) | A10 (Connectivity §4.4) | None |
| L-10 | Device publishing telemetry directly to the ingestion point | Replace | 24-hour encrypted non-volatile local store-and-forward buffer; live reading sent first upon reconnect, background buffer drain, AWS IoT PUBACK release | A10 (Connectivity §4.3) | D08, D09 |
| L-11 | Collection center without LAN or WAN link | Replace | Standardized center pattern: 7 isolated VLANs (101-120), edge firewall (10.4x.20.1), dual IPsec tunnels over 100 Mbps broadband + cellular backup to Transit Gateway hub (ASM-083) | A11 / A07 (Collection-Center Network) | D05 |
| L-12 | Port of Buenaventura without LAN or WAN link | Replace | Zero Federation infrastructure deployed at port (case statement mandate); transporter app records maritime delivery &amp; custody handoff via commercial mobile carrier sync | A10 / A16 (Connectivity §10) | None |
| L-13 | Dependence on a mobile access network generation | Retire | Dependence on a mobile access network generation switched off before or during the pilot (ASM-041). The case statement names no generation (ASM-041); pure IP bearer without generation dependency is enforced (ASM-072) | Case statement p. 2 / ASM-041, ASM-072 | None |
| L-14 | 2020 prices | Retire | Case statement explicitly states 2020 prices are not a valid quote (p. 1); all cloud infrastructure, devices, and operations priced at current 2026 market rates under open decision PD-017 | Case statement p. 1 / A23, PD-017 | None |
| L-15 | 2020 devices (none named in the case statement) | Retire | Case statement states 2020 devices are not a valid quote (p. 1) and names no specific device; replaced by functional device classes defined by A10 and priced by A23 | Case statement p. 1 / A10, A23 | None |
| L-16 | 2020 cloud services (none named in the case statement) | Retire | Case statement states 2020 cloud services are not a valid quote (p. 1) and names none; cloud architecture chosen and justified anew under ADR-001 (AWS primary in us-east-1, Azure secondary prepared) | Case statement p. 1 / ADR-001 | None |
| L-17 | The 2020 architecture as a whole | Retire | Case statement mandates: "No se replica la arquitectura de 2020" (p. 1); superseded completely by the 2026 architecture document and complete architecture diagram (A19) | Case statement p. 1 / A19 | None |
| L-18 | 2020 design manual | Retire | Case statement mandates: "No se hereda un manual de 2020" (p. 7); Challenge Point 7 establishes a new design system with 4 standardized badges for the 7 sack lifecycle states | Case statement p. 7 / Epic 7, PD-154 | None |

#### 2026 Architectural Foundations Panel

1. **Primary Cloud & Multi-Account Architecture (ADR-001)**: AWS Organizations with us-east-1 primary Region. Three workload accounts (dev, qa, pdn) with isolated IAM roles and network boundaries (ASM-060).
2. **Prepared Secondary Cloud Target (ADR-001, ASM-035)**: Azure RFC 1918 supernet 10.88.0.0/13 reserved as export target, undeployed in pilot. Avoids multi-cloud operational cost while preserving cloud portability.
3. **Plan-Only Modular Infrastructure as Code (ADR-003, D01-D03)**: Terraform root stacks consuming public tagged modules. S3 remote state with native locking, automated GitHub Actions CI/CD plan validation.
4. **Standardized Functional Device Classes & 2026 Pricing (A10, A23)**: Vendor-neutral device specifications (field telemetry nodes, drone kit, mobile outbox). Current 2026 market pricing for 200 farms under open decision PD-017.
5. **Zero-Trust Identity & 5-Year Compliance Audit Pipeline (D06, ASM-067)**: X.509 device certificates (mTLS), Cognito User Pools (staff MFA / field passwordless). Dual-layer audit with S3 Object Lock 5-year compliance archive.
6. **Modern Accessible Design System (Epic 7, PD-154)**: Modern responsive UI replacing legacy manual. 4 accessible status badges mapping the 7 coffee sack lifecycle states with color and icon redundancy.

### Source documents

- `aroma-pacifico-architecture/docs/legacy-2020.md`: all sections (decision values, exclusion of 2020 quotes, keep/retire/replace table L-01 to L-18, cross-references).
- `aroma-pacifico-architecture/docs/architecture.md`: Section 13 ("The 2020 pilot: what is kept, changed and retired"), Section 14 ("Diagram boxes D01 to D13 and traceability to the IaC").
- `aroma-pacifico-architecture/docs/case/case-statement.pdf`: pages 1, 2, 4, 7.
- `aroma-pacifico-architecture/docs/decision-log.md`: ASM-001, ASM-030, ASM-035, ASM-040, ASM-041, ASM-058, ASM-060, ASM-061, ASM-067, ASM-072, ASM-073, ASM-077, ASM-083, ASM-135, ASM-141, ASM-150, PD-004, PD-006, PD-007, PD-008, PD-017, PD-051, PD-071, PD-072, PD-073, PD-081, PD-122, PD-154, PD-161, ADR-001, ADR-002, ADR-003.

### Boxes D01-D13 shown

- `D04`: VPC tiers and private endpoints of the pattern environment (public query front door CloudFront + AWS WAF + Lambda URL for L-06).
- `D05`: Transit Gateway hub association and pattern collection center network (Cauca, Huila, Tolima for L-02 and L-11).
- `D08`: Reusable farm module pattern (crop & drying sensor nodes, 24-hour store-and-forward local buffer for L-03 and L-10).
- `D09`: Telemetry ingestion pattern module (AWS IoT Core, topic rules, DynamoDB telemetry store for L-03 and L-10).
- `D10`: Event-driven inference functions pattern (threshold evaluation, alert confirmation, model invocation for L-03 and L-07).
