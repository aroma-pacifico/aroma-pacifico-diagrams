# Legacy Diagrams

This folder contains the architectural legacy traceability diagrams for the Aroma Pacífico coffee traceability platform.

## 2020 Legacy Mapping: Keep, Retire and Replace

`legacy-2020.drawio` is the editable, uncompressed UTF-8 Draw.io XML source. `legacy-2020.svg` is the vector publication export with embedded diagram data, and `legacy-2020.png` is the 2400-pixel high-resolution preview export.

### What the diagram shows

- **Full 18-element legacy traceability matrix**: Every element of the 2020 pilot design identified in the case statement (rows L-01 through L-18 of `docs/legacy-2020.md`) is mapped to its authoritative 2026 architectural outcome:
  - **Kept requirements (L-01 to L-08)**: 2020 business and functional capabilities that remain in force for 2026:
    - `L-01`: Traceability chain from farm to buyer through wet mill, overland transport, collection center, and port of Buenaventura (A16). The 60 kg sack is the immutable traced unit with a 5-year compliance audit trail (S3 Object Lock, ASM-067).
    - `L-02`: Geographic scope covering Cauca, Huila, and Tolima with one regional collection center per department (A07, B05 / D05). Centers allocated dedicated RFC 1918 supernet `10.40.0.0/13` (Cauca `10.41.0.0/16`, Huila `10.42.0.0/16`, Tolima `10.43.0.0/16`) with room for a 4th department at `10.44.0.0/16` (ASM-030).
    - `L-03`: Crop and solar drying monitoring with alerts to the coffee grower (A10, B08-B10 / D08, D09, D10). Two nodes per farm (crop 6 readings/day, drying 24 readings/day while active), AWS IoT Core mTLS ingestion, DynamoDB telemetry store, and alert deduplication by episode (ASM-135) dispatched via SMS and mobile push.
    - `L-04`: Harvest recording and packing in 60 kg sacks (A16). Measured digital scale weight captured at wet mill registration with 60 kg reference (ASM-150) and cryptographic sack ID code generation.
    - `L-05`: Overland journey tracking to collection centers and Buenaventura port (A10, A16). Two trip legs tracked via transporter app with 5-minute GNSS fixes (`transport_position`) and explicit custody transitions (`sack_custody`).
    - `L-06`: Public sack lookup by code for farmer, center, and buyer (A09, A16, B04 / D04). Unauthenticated public front door via Amazon CloudFront, AWS WAF, and Lambda function URL (ASM-058, PD-051) strictly limited to the buyer view (variety, cup score, certifications; farmer cédula and internal trace withheld).
    - `L-07`: Five drone flight missions preserved (ADR-002, A15): plot delimitation, alert confirmation, targeted spot fumigation, treatment verification, and harvest estimation. Governed under combined flight mode (contracted operator, ASM-141) with agronomist MFA authorization and S3 telemetry audit.
    - `L-08`: Three target application API contracts (A09): farmer mobile app, transporter mobile app, and public web lookup. Case statement establishes apps are planned but not constructed in challenge scope; 2026 delivers REST/GraphQL APIs, Cognito authentication pools (staff MFA / field passwordless), and ingestion backend contracts.
  - **Replaced elements (L-09 to L-12)**: 2020 connectivity assumptions and missing infrastructure replaced with resilient cloud-native designs:
    - `L-09`: Assumption of continuous cellular coverage across farms replaced with explicit no-signal handling for every data flow and satellite IP bearer for out-of-coverage farms (A10, ASM-073, PD-081).
    - `L-10`: Direct unbuffered telemetry publishing replaced with a 24-hour encrypted non-volatile local store-and-forward buffer (A10, B08, B09 / D08, D09). Live readings transmit first on reconnect; oldest readings drain in background; AWS IoT Core PUBACK releases buffer entries.
    - `L-11`: Collection centers without LAN or WAN replaced with standardized center network (A11, A07, B05 / D05): 7 isolated VLANs (101-120), edge perimeter firewall (`10.4x.20.1`), and dual IPsec VPN tunnels over 100 Mbps broadband + cellular backup to AWS Transit Gateway hub (ASM-083).
    - `L-12`: Port of Buenaventura without LAN or WAN replaced with zero Federation infrastructure at the port (case statement mandate). Transporter app records maritime delivery and custody handoff via commercial mobile carrier directly to public ingestion API (A10, A16, ASM-077).
  - **Retired elements (L-13 to L-18)**: Legacy elements dropped without direct replacement, crossed out with red strike line and strikethrough typography, accompanied by explicit justification citing case statement mandates:
    - `L-13`: Dependence on 2G/3G mobile generation retired due to carrier generation sunset (ASM-041, ASM-072); pure packet-switched IP bearer (4G LTE / Satellite IP) enforced.
    - `L-14`: 2020 prices retired under case statement mandate ("Precios de 2020 no son cotización válida"); priced at current 2026 market rates by A23 (PD-017).
    - `L-15`: 2020 device models retired under case statement mandate ("Dispositivos de 2020 no son cotización válida"); replaced by standardized functional device classes (field nodes, ground station, mobile outbox) specified by A10 and priced by A23.
    - `L-16`: 2020 cloud services retired under case statement mandate ("Servicios de nube de 2020 no son cotización válida"); AWS primary cloud (`us-east-1`) and Azure prepared target chosen anew under ADR-001.
    - `L-17`: The 2020 architecture as a whole retired under case statement mandate ("No se replica la arquitectura de 2020"); replaced by modern event-driven serverless architecture documented in A19.
    - `L-18`: 2020 design manual retired under case statement mandate ("No se hereda un manual de 2020"); replaced by Challenge Point 7 new design system (Epic 7, PD-154).
  - **2026 Architectural Foundations**: Right-hand panel detailing the modern infrastructure paradigm that supersedes the legacy system:
    1. Primary cloud & multi-account architecture (AWS Organizations, `dev`/`qa`/`pdn` workload accounts in `us-east-1`, ADR-001).
    2. Prepared secondary cloud target (Azure `10.88.0.0/13` export target, undeployed in pilot, ASM-035).
    3. Plan-only modular Infrastructure as Code (Terraform tagged modules, S3 remote state with native locking, ADR-003, D01-D03).
    4. Standardized functional device classes and 2026 pricing schedule (A10, A23, PD-017).
    5. Zero-trust identity and 5-year compliance audit pipeline (X.509 mTLS device certs, Cognito pools, S3 Object Lock compliance archive, D06, ASM-067).
    6. Modern accessible design system (Epic 7, PD-154) with 4 status badges for 7 sack lifecycle states.

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
