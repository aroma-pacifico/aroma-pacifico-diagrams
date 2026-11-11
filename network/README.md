# Network Diagrams

This folder contains the network architecture and addressing diagrams for Aroma Pacífico.

## Cloud and Site Addressing Plan

`addressing-plan.drawio` is the editable, uncompressed Draw.io XML source. `addressing-plan.svg` is the vector publication export with embedded diagram data, and `addressing-plan.png` is the 2400-pixel preview export.

### What the diagram shows

- **Top-level RFC 1918 IPv4 partition**: Three non-overlapping /13 supernets separating on-premises collection center space (`10.40.0.0/13`), AWS primary cloud supernet (`10.80.0.0/13`, `us-east-1`), and secondary Azure cloud space (`10.88.0.0/13`, prepared and undeployed).
- **Mathematical non-overlap verification**: Bounds checking proving strict separation between center space (`10.47.255.255 < 10.80.0.0`), AWS supernet (`10.87.255.255 < 10.88.0.0`), and distinct /16 VPC blocks per environment (`10.80.0.0/16` dev, `10.81.0.0/16` qa, `10.82.0.0/16` pdn).
- **AWS VPC subnet hierarchy and tier layout**: Detailed subnet structure for the eight functional tiers plus transit and reserved tiers across three active Availability Zones (`az1`, `az2`, `az3`) and one reserved slot (`az4`):
  - `public_query` (`10.E.0.0/20`, /22 subnets): Public routing via IGW; front door is CloudFront + AWS WAF, no ALB, no NAT GW (ASM-127, PD-051). Created empty in pilot.
  - `telemetry_ingress` (`10.E.16.0/20`, /22 subnets): Private routing; farm nodes publish to regional AWS IoT Core public endpoint. No interface endpoint created in pilot (CL-TEL-04, ASM-032). Tier empty and denied.
  - `drone_object_ingress` (`10.E.32.0/20`, /22 subnets): Private routing; reserved tier with no active path (ASM-140, CC-DRN-02). Ground station uploads to `application` and `object_storage` endpoints.
  - `application` (`10.E.48.0/20`, /22 subnets): Private routing; Lambda function network interfaces, API Gateway and Cognito interface endpoints.
  - `data` (`10.E.64.0/20`, /22 subnets): Private routing, no internet route; managed database subnet groups across AZs (DynamoDB is regional managed on-demand, ASM-061).
  - `object_storage` (`10.E.80.0/20`, /22 subnets): Private routing; Amazon S3 interface VPC endpoints reachable from collection centers through TGW.
  - `administration` (`10.E.96.0/20`, /22 subnets): Private routing; interface endpoints for SSM Session Manager, KMS, IAM, CloudWatch Logs, Parameters without bastion hosts.
  - `Reserved Tiers` (`10.E.112.0 to 10.E.239.255`, eight /20 blocks, 32,768 addresses): Unallocated private addresses per environment for future architecture evolution without moving existing subnets.
  - `transit` (`10.E.240.0/20`, /28 slots): Private routing; Transit Gateway hub attachments routing `10.40.0.0/13` to centers.
- **Collection center LAN pattern (`10.4x.Z.0/24`)**: Identical pattern applied across Cauca (`4x = 41`), Huila (`4x = 42`), Tolima (`4x = 43`), and future fourth department (`4x = 44`, ASM-030):
  - Host layout: `.1` firewall gateway, `.2` to `.49` static and DHCP reservations, `.50+` dynamic DHCP pool (ASM-083).
  - Seven zones: Gerencia (VLAN 101, `cc-management`, EAP-TLS), Soporte (VLAN 102, `cc-support`, EAP-TLS, contracted drone ground station fixed at `10.4x.2.10`), Logística (VLAN 103, `cc-logistics`, EAP-TLS, 2 desktop reception posts), Transporte (VLAN 104, `cc-transport`, WPA3-SAE, client isolation, local breakout), Visitantes (VLAN 105, `cc-guest`, WPA3-SAE, client isolation, local breakout, no routing to logistics or management), Servidores locales (VLAN 110, wired only, switches `.2-.9`, APs `.20-.49`, printers `.50-.59`), and Enlace a la nube (VLAN 120, edge ports, firewall `10.4x.20.1`, ISP `.2`, backup cellular `.3`).
- **WAN interconnection**: Dual IPsec VPN tunnels connecting the center firewall (`10.4x.20.1`) over 100 Mbps internet (90 Mbps usable payload + 10 Mbps encapsulation) and cellular backup to the AWS Transit Gateway hub in the network account.
- **Scaling path inset (200 to 1,000 farms)**: Demonstrates compliance with the technical constraint requiring scale from 200 to 1,000 farms without redesigning addressing or the ingestion point:
  - Identity-based ingestion over cellular/satellite to AWS IoT Core (ASM-032); zero private cloud IP addresses consumed.
  - Telemetry load scaling from 3,600 readings/day (~0.04/s) at 200 farms to 18,000 readings/day (~0.21/s) at 1,000 farms (ASM-036).
  - Pessimistic headroom analysis: even if every farm consumed one private IP, 1,000 farms fit into a single AZ slot of `telemetry_ingress` (1,019 usable IPs) and use only 32.7% of the tier's active capacity (3,057 IPs).

### Source documents

- `aroma-pacifico-architecture/docs/addressing-plan.md`: all sections (purpose, reserved collection-center space, top-level allocation, subnet layout, scaling from 200 to 1,000 farms, non-overlap verification).
- `aroma-pacifico-architecture/docs/collection-center-network.md`: section 2 (one pattern, three centers) and section 3 (zones, VLANs, SSIDs, host addressing, infrastructure zones, drone ground station at `10.4x.2.10`).
- `aroma-pacifico-architecture/docs/architecture.md`: section 14 (diagram boxes D01 to D13 and traceability to IaC).
- `aroma-pacifico-architecture/docs/case/case-statement.pdf`: point 3 (collection center network inventory, non-overlapping cloud addressing, minimal subnets, 200 to 1,000 farm scaling constraint).

### Diagram boxes shown

- **D04** (Track B issue aroma-pacifico/aroma-pacifico-roadmap#24 / B04): VPC and tiers of the pattern environment (`qa`), defining CIDRs, subnet blocks, AZ distribution, and routing boundaries.
- **D05** (Track B issue aroma-pacifico/aroma-pacifico-roadmap#25 / B05): Association of the environment with the pattern collection center, WAN interconnection, and Transit Gateway routing.

---

## End-to-End Connectivity Architecture (`connectivity.drawio`)

Architecture diagram of all seven connectivity flows from field devices, wet mills, drones, transporters, and collection centers through WAN links, IPsec tunnels, the transit network hub, and VPC tiers to cloud services.

- **Primary Source:** [`connectivity.drawio`](connectivity.drawio)
- **Vector Publication Export:** [`connectivity.svg`](connectivity.svg)
- **High-Resolution Preview:** [`connectivity.png`](connectivity.png)

### What it shows

- **Flow 1 (Farm Crop Telemetry):** Autonomous solar/battery crop nodes publishing soil probe readings (NPK, moisture, temperature) every 4 hours via MQTT 3.1.1/5 over TLS 1.2+ with per-node mutual X.509 certificates to the regional AWS IoT Core public endpoint (`us-east-1`). Farm traffic completely bypasses collection centers and VPC subnets.
- **Flow 2 (Farm Out of Coverage):** Dual-mode terrestrial cellular and non-terrestrial satellite modules (contact interval <= 4 hours) transmitting MQTT over TLS to the same regional IoT Core endpoint without cloud re-addressing or redesign.
- **Flow 3 (Wet Mill):** Drying node publishing hourly coffee moisture and grain temperature via MQTT over TLS to IoT Core. Farmer smartphone registering 60 kg sacks (weight, sack ID, farmer ID, timestamp) via HTTPS to CloudFront / Public Application API backed by the `application` tier.
- **Flow 4 (Contracted Drone Operator):** Contracted operator drone and ground station (ADR-002) with preloaded missions. Ingress from Collection Center via Support LAN (`10.4x.2.10/32`) transits the Center Firewall, IPsec VPN, TGW Hub, and `transit` tier to interface endpoints in `application` (TCP 443) and `object_storage` (Amazon S3 interface endpoint, TCP 443). Municipal seat operations upload directly to regional S3 via pre-signed multipart URLs. The `drone_object_ingress` tier (`10.82.32.0/20`) is reserved and holds no endpoint in the pilot.
- **Flow 5 (Transporter en Route & Yard):** Transporter smartphone recording GNSS fixes every 5 minutes and custody events (`transport_position` and `sack_custody`) into an encrypted 7-day outbox. Uplinks via cellular to Public Application API. Optional collection center yard access via `cc-transport` SSID (`10.4x.4.0/24`) uses local breakout directly to the public internet, never entering the IPsec tunnel.
- **Flow 6 (Collection Center Operations):** Three identical collection centers (Cauca `10.41.0.0/16`, Huila `10.42.0.0/16`, Tolima `10.43.0.0/16`) summarized as `10.40.0.0/13`. Center Perimeter Firewall in Cloud Link zone (`10.4x.20.1`) terminates dual IPsec tunnels over 100 Mbps WAN with cellular backup to the AWS Transit Gateway Hub in the Network Account. Logistics posts (`10.4x.3.0/24`) reach the `application` tier privately (`CC-LOG-01`). Code-sheet printers (`10.4x.10.50-.59`) stay strictly on the local LAN (`CC-LOG-02`). Cloud administration (`10.82.96.0/20`) administers center devices via session administration (`CC-ADM-01`).
- **Flow 7 (Port of Buenaventura):** Transporter records sack delivery handoff (`delivered_at_port`, accepted or rejected, receiver receipt reference) via commercial cellular data to the Public Application API. Zero Federation equipment deployed at the port.
- **Secondary Cloud:** Azure Virtual Network `10.88.0.0/13` marked as prepared, not deployed (ADR-001).

### Source Documents and Sections

- `aroma-pacifico-architecture/docs/connectivity.md`:
  - Section 2: Design principles (Identity not address, Bearer neutrality, Original timestamp and idempotency, Release on acknowledgment, End-to-end encryption)
  - Section 3: Summary table of all flows
  - Section 4: Farm equipment, exit protocols, 24-hour buffer, out-of-coverage satellite mechanism
  - Section 5: Wet mill drying telemetry and sack registration
  - Section 6: Drone operations, preload, dual ingress paths, reserved tier rationale
  - Section 7: Transporter 5-minute interval, persisted events, outbox sizing
  - Section 8: Crop alert latency budget and SLO tie
  - Section 9: Collection center WAN, IPsec tunnels, routing, backup links
  - Section 10: Port of Buenaventura delivery event
  - Section 11: Mobile network generation neutrality (LTE-based LPWA, no 2G/3G)
- `aroma-pacifico-architecture/docs/collection-center-network.md`:
  - Section 2: One pattern, three centers (Cauca 41, Huila 42, Tolima 43)
  - Section 3: Zones, VLANs (101-105, 110, 120), SSIDs, WPA3-Enterprise 802.1X, addressing
  - Section 4: Drone ground station on Support LAN (`10.4x.2.10/32`) and network device management
- `aroma-pacifico-architecture/docs/firewall-policy.md`:
  - Section 4: Center perimeter firewall rules (`CC-GST-01` to `CC-DEF-01`)
  - Section 5: Cloud security controls and prefix lists (`CL-LNK`, `CL-APP`, `CL-DRN`, `CL-ADM`)
- `aroma-pacifico-architecture/docs/architecture.md`:
  - Section 14: Diagram boxes D01 to D13 and traceability to IaC

### Traceability Badges (D01-D13)

- **[D04] VPC and Tiers:** Production VPC (`10.82.0.0/16`) and tiers (`transit`, `application`, `object_storage`, `administration`, `data`, `telemetry_ingress`, `drone_object_ingress`, `public_query`, `egress`).
- **[D05] Collection Center Association:** `10.40.0.0/13` summary route, dual IPsec Site-to-Site VPN tunnels over 100 Mbps WAN with cellular backup, AWS Transit Gateway Hub in the Network Account.
- **[D07] Pattern Firewall Rules:** Center perimeter firewall rules (`CC-*`) and cloud security groups, prefix lists, and endpoint policies (`CL-*`).
- **[D08] Reusable Farm Module:** Pattern farm crop and drying nodes, multi-network SIM, dual-mode satellite module, and 24-hour non-volatile buffer.
- **[D09] Telemetry Ingestion Module:** Regional AWS IoT Core public endpoint (`us-east-1`), mutual X.509 certificate authentication, and idempotent DynamoDB store.
- **[D10] Event-Driven Inference Pattern:** Threshold Lambda, SQS threshold queue, and Amazon SNS alert topic.
- **Azure:** `10.88.0.0/13` prepared, not deployed secondary cloud export target (ADR-001).

---

## No-Signal Handling and Crop Alert Sequence (`no-signal-sequence.drawio`)

Sequence diagram modeling offline buffering, reconnection replay ordering, idempotent deduplication, and the 7-hop crop alert delivery latency budget.

- **Primary Source:** [`no-signal-sequence.drawio`](no-signal-sequence.drawio)
- **Vector Publication Export:** [`no-signal-sequence.svg`](no-signal-sequence.svg)
- **High-Resolution Preview:** [`no-signal-sequence.png`](no-signal-sequence.png)

### What it shows

- **Part 1: The No-Signal Case (Offline Buffering, Reconnection, Ordered Replay & Deduplication):**
  - **Normal Operation:** Direct telemetry ingestion, DynamoDB persistence, and MQTT PUBACK.
  - **Signal Loss:** Link drop (`link_state: offline`). Sensor readings and GNSS fixes continue at regular cadence. Events stored in local encrypted non-volatile buffer with original timestamp, sequence number, and idempotency key (`node_id` + `reading_timestamp` or `event_id`).
  - **Buffer Constraints:** Farm telemetry ring buffer enforces 24-hour maximum retention (ASM-074); unacknowledged records older than 24 hours are evicted and `drop_counter` is incremented. Phone outbox retains 7 days of fixes (custody events never evicted, ASM-076, ASM-079). Logistics outbox retains reception records until ACK.
  - **Reconnection & Ordered Replay:**
    1. **Live Reading First:** Fresh reading taken at reconnect is published immediately so live out-of-range values never wait behind historical buffer backlog.
    2. **Oldest First Drain:** Buffered history drains in chronological order (oldest first). First message carries `drop_counter` and sequence number range (gap report).
  - **Idempotent Deduplication:** Primary key checks in DynamoDB and database stores ensure retransmitted packets (e.g. dropped PUBACK) are absorbed without duplicate alerts or corrupt records.
  - **Release on Acknowledgment:** Buffer entries are deleted only after verified cloud acknowledgment (MQTT PUBACK / HTTP 2xx / S3 checksum).
  - **Drained Reading Evaluation:** Evaluated by Threshold Lambda at drain time; alert delay is measured against original timestamp.
- **Part 2: Crop Alert Latency Budget (< 5 Minutes / 300 Seconds):**
  - Step-by-step breakdown across the seven hops defined in `docs/connectivity.md` Section 8.2:
    - **Hop 1:** Sensor reading taken to message ready on node: **10 s target** (no local evaluation; formatted immediately).
    - **Hop 2:** Radio wake, cellular network attach, TLS 1.2+ mutual handshake: **60 s target** (largest field hop; verified in satellite procurement).
    - **Hop 3:** MQTT publish (QoS 1) to broker PUBACK at IoT Core: **20 s target** (includes internet round-trip and transient retry).
    - **Hop 4:** Topic rule to SQS threshold queue invocation: **10 s target** (decouples ingestion from compute scaling).
    - **Hop 5:** Threshold Lambda evaluation & persistence: **20 s target** (reads data-driven ranges, writes alert to DB, publishes to SNS).
    - **Hop 6:** SNS topic to notification channel provider: **30 s target** (HTTPS dispatch to push gateway / SMS aggregator).
    - **Hop 7:** Notification provider to farmer smartphone: **60 s target** (cellular delivery of push notification / SMS).
  - **Timing Budget:** Total target latency is **210 s** (3.5 minutes), providing a **90 s safety margin** below the **300 s hard case statement limit** ("menos de 5 minutos").
  - **Traceability to A21 SLO:** Trace points at Hops 1, 3, 5, 6, and 7 allow attributing any SLO breach to the exact failing hop.

### Source Documents and Sections

- `aroma-pacifico-architecture/docs/connectivity.md`:
  - Section 4.3: Store-and-forward up to 24 hours (buffer specification, drain rules, gap report)
  - Section 6: Drone storage without time limit, buffer rules, and flight closure
  - Section 7: Transporter 5-minute interval, persisted events, outbox retention
  - Section 8.1: Route available definition
  - Section 8.2: Latency budget per hop table (Hops 1 to 7)
  - Section 8.3: Tie to the alert-delivery SLO (SLI definition and trace points)
- `aroma-pacifico-architecture/docs/decision-log.md`:
  - `ASM-011`: Original reading timestamp and idempotency key
  - `ASM-074`: Node buffer specification (24 h retention, live first, then oldest, PUBACK release)
  - `ASM-075`: Per-hop alert budget values as design targets
  - `ASM-076`: Transporter position without data, 7-day outbox capacity
  - `ASM-079`: App outbox releases on acknowledgment without time-based eviction
  - `PD-081`: Satellite IP service attach time verification
  - `PD-082`: A21 adoption of SLI, trace points, and alert budget
  - `PD-085`: Farmer alert notification channel selection within Hop 7 budget

### Traceability Badges (D01-D13)

- **[D08] Reusable Farm Module:** Field node offline buffering, 24-hour ring buffer, and ordered drain logic.
- **[D09] Telemetry Ingestion Module:** AWS IoT Core QoS 1 acknowledgment, buffer release trigger, and idempotent DynamoDB store.
- **[D10] Event-Driven Inference Pattern:** SQS threshold queue, Threshold Lambda evaluation, and Amazon SNS notification dispatch.

## Collection Center Network Pattern (`collection-center.drawio`)

### What It Shows

The diagram draws one collection center as a site boundary. The same pattern applies to the three centers: Cauca (`10.41.0.0/16`), Huila (`10.42.0.0/16`) and Tolima (`10.43.0.0/16`), inside the reserved `10.40.0.0/13`, with `4x` standing for 41, 42 or 43. Inside the boundary are the seven zones of the case statement with their VLAN, network, SSID and gateway, the perimeter firewall that routes and filters every inter-zone flow, the two WAN links of Enlace a la nube, the logistics reception posts and the code-sheet printers.

Outside the boundary are the three destinations the center reaches. The two IPsec tunnels end on the Site-to-Site VPN of the hub in the AWS network account, whose transit gateway attaches the production VPC `pdn` (`10.82.0.0/16`). Inside that VPC the diagram shows the `transit`, `administration`, `application` and `object_storage` tiers and the reserved `drone_object_ingress` tier. The local internet breakout serves the AWS IoT Core endpoint, the public front doors and non-operational web traffic.

Green solid arrows are allowed flows and red dashed arrows are denied flows; each carries its rule ID from `firewall-policy.md` section 4 and its port or the word DENY. The arrows show the policy intent of `collection-center-network.md` section 6: staff and logistics reach `application` through the tunnel, only the ground station `10.4x.2.10/32` reaches `object_storage` and is denied the reserved drone tier, the cloud `administration` tier manages the devices, printing stays on the LAN, and Transporte and Visitantes live in an internet-only routing context with an explicit deny toward Logística and Gerencia. The Soporte printing arrow crosses the two deny lanes; the crossings are drawn as line jumps.

### Zone Layout and Addressing

### Zone Layout and Addressing

| Zone | VLAN | Network CIDR | SSID | Authentication | Gateway | Planned Clients | DHCP Pool / Lease | Traffic Class / QoS |
|---|---|---|---|---|---|---|---|---|
| Enlace a la nube | 120 | `10.4x.20.0/24` | Wired only | Static edge ports | Edge: `10.4x.20.1` | Primary WAN (`.2`), Cellular (`.3`) | None (static) | WAN Uplink (100 Mbps symmetric) |
| Gerencia | 101 | `10.4x.1.0/24` | `cc-management` | WPA3-Enterprise (802.1X EAP-TLS) | `10.4x.1.1` | 2 desktops, 5 phones (7 total) | `10.4x.1.50-.69` (20 hosts, 12 h) | Staff Class (Priority 4, 28/7 Mbps gtd) |
| Soporte | 102 | `10.4x.2.0/24` | `cc-support` | WPA3-Enterprise (802.1X EAP-TLS) | `10.4x.2.1` | 2 desktops, 5 phones + Drone station (`.10`) | `10.4x.2.50-.69` (20 hosts, 12 h) | Staff Class (P4) & Drone Class (P3, 5/20 Mbps gtd) |
| Logística | 103 | `10.4x.3.0/24` | `cc-logistics` | WPA3-Enterprise (802.1X EAP-TLS) | `10.4x.3.1` | 2 reception desktops, 20 handheld scanners (22 total) | `10.4x.3.50-.99` (50 hosts, 12 h) | Logistics Class (Priority 2, 44/11 Mbps gtd) |
| Transporte | 104 | `10.4x.4.0/24` | `cc-transport` | WPA3-Personal (SAE, rotated per harvest) | `10.4x.4.1` | 20 transporter phones (20 total) | `10.4x.4.50-.89` (40 hosts, 4 h) | Transport App (P4, 2/1M gtd) & Non-op (P5, capped) |
| Visitantes | 105 | `10.4x.5.0/24` | `cc-guest` | WPA3-Personal (SAE, weekly rotated) | `10.4x.5.1` | 10 guest phones (10 total) | `10.4x.5.50-.69` (20 hosts, 1 h) | Non-operational (P5, capped at 10/2.5M, 2/0.5M/client) |
| Servidores locales | 110 | `10.4x.10.0/24` | Wired only | Static & DHCP MAC reservation | `10.4x.10.1` | Network devices (`.2-.49`), Printers (`.50-.59`) | Reservations only (no dynamic pool) | Network Control (Priority 1, 2/2 Mbps gtd) |


### Devices Behind the Zone Boxes

The zone boxes on the diagram carry one or two lines each. The detail they summarize is listed here, from `collection-center-network.md` sections 3.4, 4, 5 and 9.

| Device or group | Zone | Address | Detail |
|---|---|---|---|
| Perimeter firewall, one per center (ASM-088) | Enlace a la nube (edge), Servidores locales (management) | Edge `10.4x.20.1`, management `10.4x.10.1`, gateway `10.4x.Z.1` of each zone | Stateful zone policy between VLAN subinterfaces; two IPsec tunnels; second WAN interface for the cellular backup; internet-only routing context for VLANs 104 and 105; DHCP server and DNS resolver; RADIUS for EAP-TLS with local revocation checking; traffic classes with guarantees and caps. |
| Primary internet service equipment | Enlace a la nube | `10.4x.20.2` | 100 Mbps symmetric WAN, planned at 90 Mbps with 10 Mbps for IPsec overhead and control (ASM-085). |
| Cellular backup router, if separate | Enlace a la nube | `10.4x.20.3` | Carries the same tunnels when the primary fails; non-operational traffic is suspended (ASM-078, ASM-086). |
| Managed PoE switch | Servidores locales | `10.4x.10.2` to `.9` | 802.1Q trunk from the firewall, 802.1X on access ports, MAC-based port security for printers. |
| Wi-Fi access points | Servidores locales | `10.4x.10.20` to `.49`, DHCP reservation | Five SSIDs per radio, WPA3-Enterprise and WPA3-Personal, client isolation per SSID; the count comes from the site survey (PD-092). |
| Code-sheet printers | Servidores locales | `10.4x.10.50` to `.59`, static | Two to start, one per reception post; print jobs from Logística (CC-LOG-02) and from Gerencia and Soporte (CC-STF-02) never cross the WAN; a printer never initiates traffic (CC-ADM-04). |
| Reception posts | Logística | `10.4x.3.0/24` | Two desktops wired to Logística ports with 802.1X; the operator signs in with MFA in the application; an encrypted outbox keeps receptions recorded while the WAN is down (ASM-079). |
| Logistics phones | Logística | DHCP pool `10.4x.3.50` to `.99`, lease 12 h | Twenty phones on `cc-logistics` that read sack codes in the yard. |
| Drone ground station | Soporte | `10.4x.2.10/32`, DHCP reservation (ASM-082) | Device of the contracted drone operator (ADR-002); reaches `application` (CC-DRN-01) and `object_storage` (CC-DRN-03) through the tunnel and AWS IoT Core through the local breakout (CC-DRN-04); denied the reserved `drone_object_ingress` tier (CC-DRN-02, ASM-140). |
| Transporter phones | Transporte | DHCP pool `10.4x.4.50` to `.89`, lease 4 h | Twenty phones; reach the public API front door (CC-TEL-02) and non-operational web (CC-GST-05). |
| Visitor phones | Visitantes | DHCP pool `10.4x.5.50` to `.69`, lease 1 h | Ten phones; non-operational web only (CC-GST-05). |

Device models and prices (PD-095) and the number of access points and switch ports (PD-092) are not fixed by the design, and the diagram says so in a note.

### Inter-Zone Policy Intent Matrix

The table summarizes the perimeter firewall rules of `firewall-policy.md` section 4. The diagram draws the rules that define the zone policy intent as arrows; the rest live only in this table.

| Rule ID | Source Zone | Destination Zone | Port / Protocol | Action | Description and Justification |
|---|---|---|---|---|---|
| `CC-GST-01` | Visitantes (105) | Logística (103), Gerencia (101) | Any | Deny | Case statement point 3: guest VLAN never routes to logistics or management. Internet-only routing context. |
| `CC-GST-02` | Transporte (104) | Logística (103), Gerencia (101) | Any | Deny | Transporters are non-staff; denied to user LAN and management zones. |
| `CC-GST-03` | Visitantes (105), Transporte (104) | Private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) | Any | Deny | Isolated from all internal subnets, other centers, cloud tiers, and VPN tunnels. |
| `CC-GST-04` | Visitantes (105), Transporte (104) | Internet | TCP 25 (SMTP) | Deny | Prevents guest phones from relaying outbound spam through center breakout. |
| `CC-GST-05` | Visitantes (105), Transporte (104) | Internet | Any other | Allow | Non-operational internet breakout, capped per zone and client (2 / 0.5 Mbps). |
| `CC-BAS-01` | Client Zones (101-105) | Gateway (`10.4x.Z.1`) | UDP 67 (DHCP), UDP/TCP 53 (DNS), UDP 123 (NTP) | Allow | Base gateway services served locally by firewall subinterfaces. |
| `CC-BAS-02` | Switches / APs (110) | Firewall (`10.4x.10.1`) | UDP 1812, 1813 (RADIUS) | Allow | 802.1X EAP-TLS authentication requests validated against local RADIUS on firewall. |
| `CC-BAS-03` | Switches / APs (110) | Firewall (`10.4x.10.1`) | UDP 514 (Syslog), UDP 162 (SNMP traps), UDP 123 (NTP) | Allow | Device logs and time stay local; the firewall exports them (CC-ADM-02). |
| `CC-BAS-04` | Firewall (`10.4x.10.1`) | Switches / APs (110) | TCP 22, TCP 443, UDP 161 (SNMPv3) | Allow | Firewall management and polling of LAN switches and wireless APs. |
| `CC-LNK-01` | Firewall primary WAN address | Tunnel peers of the center's Site-to-Site VPN | UDP 500, UDP 4500 (NAT-T) | Allow | Dual IPsec tunnels over primary 100M symmetric fiber link. |
| `CC-LNK-02` | Firewall primary WAN address | Tunnel peers of the center's Site-to-Site VPN | IP 50 (ESP) | Allow | IPsec ESP fallback if public address is provided without NAT. |
| `CC-LNK-03` | Firewall cellular backup WAN address | Tunnel peers of the center's Site-to-Site VPN | UDP 500, UDP 4500 (NAT-T) | Allow | Secondary IPsec tunnels over LTE+ backup router. |
| `CC-LNK-04` | Firewall Edge (`10.4x.20.1`) | AWS Route 53 Resolver (`10.82.96.0/20`) | UDP/TCP 53 | Allow | Conditional DNS forwarding of platform endpoint domain names to private cloud IPs. |
| `CC-LNK-05` | Firewall Edge (`10.4x.20.1`) | Provider DNS & NTP servers | UDP 123, UDP/TCP 53 | Allow | Firewall system time synchronization and external resolution for breakout zones. |
| `CC-LNK-06` | Logística (`10.4x.3.0/24`) | QA Application (`10.81.48.0/20`) | TCP 443 | Allow | Temporary acceptance testing of collection app against QA environment. |
| `CC-LNK-07` | Any address arriving through the tunnel | Any center address not matched by CC-ADM-01 | Any | Deny | The cloud initiates only management. |
| `CC-LNK-08` | Internet (WAN interfaces) | Center LAN & Firewall | Any | Deny | All unsolicited inbound traffic dropped and logged; no WAN management. |
| `CC-LNK-09` | Center LAN | Other Centers (`10.40.0.0/13`) | Any | Deny | No center-to-center path; TGW hub enforces no spoke-to-spoke routing. |
| `CC-ADM-01` | AWS Administration (`10.82.96.0/20`) | Management addresses (`10.4x.10.1`, `.2-.9`, `.20-.49`) | TCP 22 (SSH), TCP 443 (HTTPS) | Allow | Session-based administration from cloud session hosts through tunnel. |
| `CC-ADM-02` | Firewall Edge (`10.4x.20.1`) | AWS Administration (`10.82.96.0/20`) | TCP 443 | Allow | CloudWatch Logs/Metrics and IAM Roles Anywhere interface endpoints. |
| `CC-ADM-03` | Client Zones (101-105) | Management addresses and TCP 22, 443 on every firewall interface | Any | Deny | No user zone can reach management interfaces of network devices or firewall. |
| `CC-ADM-04` | Printers (`10.4x.10.50-.59`) | Any | Any | Deny | Printers never initiate outbound connections to LAN, cloud, or internet. |
| `CC-ADM-05` | Management addresses | Vendor update service host names | TCP 443 | Allow | Temporary: opened per approved maintenance window, closed at its end. |
| `CC-ADM-06` | Commissioning Laptop (`.10.60`) | Local Servers (`10.4x.10.0/24`) | TCP 22, TCP 443 | Allow | Temporary initial on-site configuration before tunnel establishment. |
| `CC-ADM-07` | Local Servers (`10.4x.10.0/24`) | Internet | Any | Deny | No general internet breakout for infrastructure devices. |
| `CC-DRN-01` | Drone Station (`10.4x.2.10/32`) | AWS Application (`10.82.48.0/20`) | TCP 443 | Allow | Mission package download and upload-grant request under pilot identity. |
| `CC-DRN-02` | Drone Station (`10.4x.2.10/32`) | AWS Drone Ingress (`10.82.32.0/20`) | Any | Deny | Reserved tier; holds no interface endpoint in pilot (ASM-140). |
| `CC-DRN-03` | Drone Station (`10.4x.2.10/32`) | AWS Object Storage (`10.82.80.0/20`) | TCP 443 | Allow | Multipart upload of flight records and imagery directly to S3 endpoint. |
| `CC-DRN-04` | Drone Station (`10.4x.2.10/32`) | AWS IoT Core (Public endpoint) | TCP 8883 (MQTT/TLS) | Allow | Kit grant topic on public endpoint via local breakout (mutual X.509 auth). |
| `CC-DRN-05` | Drone Station (`10.4x.2.10/32`) | Any other destination | Any | Deny | Drone ground station restricted strictly to its dedicated /32 rules. |
| `CC-DRN-06` | Soporte (`10.4x.2.0/24` except `.10`) | AWS Drone Endpoints | Any | Deny | Other support clients cannot access drone upload paths or S3 endpoints. |
| `CC-LOG-01` | Logística (`10.4x.3.0/24`) | AWS Application (`10.82.48.0/20`) | TCP 443 | Allow | Coffee sack intake (Part C), reception verification, and Cognito operator auth. |
| `CC-LOG-02` | Logística (`10.4x.3.0/24`) | Printers (`10.4x.10.50-.59`) | TCP 631 (IPP) | Allow | Print jobs for un-coded coffee sacks; stays local on LAN without crossing WAN. |
| `CC-LOG-03` | Logística (`10.4x.3.0/24`) | Managed OS & MDM Updates | TCP 443 | Allow | Restricted allow-list for reception workstation and scanner OS/MDM updates. |
| `CC-LOG-04` | Logística (`10.4x.3.0/24`) | Internet (other) | Any | Deny | No general internet breakout for logistics reception posts or yard scanners. |
| `CC-STF-01` | Gerencia (101), Soporte (102) | AWS Application (`10.82.48.0/20`) | TCP 443 | Allow | Center application, operational dashboard, and reports web access. |
| `CC-STF-02` | Gerencia (101), Soporte (102) | Printers (`10.4x.10.50-.59`) | TCP 631 (IPP) | Allow | Administrative print jobs; stays local on LAN. |
| `CC-STF-03` | Gerencia (101), Soporte (102) | Other Client Zones | Any | Deny | Inter-zone isolation between management, support, logistics, and guest zones. |
| `CC-STF-04` | Gerencia (101), Soporte (102) | Internet (Public IPs) | TCP 80, TCP 443, UDP 443 | Allow | Staff web access via local breakout (Staff traffic class, 28/7 Mbps gtd). |
| `CC-STF-05` | Gerencia (101), Soporte (102) | Internet (other ports) | Any | Deny | Restricts staff breakout exclusively to web ports. |
| `CC-TEL-01` | Center LAN | AWS Telemetry Ingress (`10.82.16.0/20`) | Any | Deny | Farm telemetry never crosses a center; the tier is not routed from the centers. |
| `CC-TEL-02` | Transporte (104) | Public API Front Door | TCP 443 | Allow | Transporter location updates and custody events via breakout (CloudFront/WAF). |
| `CC-PUB-01` | Client Zones (101, 102, 104, 105) | Public Sack Lookup | TCP 443 | Allow | Public sack verification front door reached via local breakout. |
| `CC-DEF-01` | Any | Any | Any | Deny | Explicit default deny; all unmatched flows dropped and logged for auditing. |


### Source Documents and Sections

| Document in `aroma-pacifico-architecture` | Sections used |
|---|---|
| `docs/collection-center-network.md` | 2 (one pattern, three centers), 3 (zones, VLANs and SSIDs), 4 (where each element connects), 5 (logistics workstations and code-sheet printing), 6 (inter-zone policy intent), 8 (traffic classes), 9 (device classes), 10 (behavior when the WAN fails) |
| `docs/firewall-policy.md` | 4 (perimeter firewall of each collection center, rules CC-GST-01 to CC-DEF-01) and 5.2 (cloud rules CL-LNK-01 to CL-LNK-08 and CL-TEL-03 for the hub and the front door) |
| `docs/addressing-plan.md` | 4 (tier blocks of the environment VPC, including `transit` `10.E.240.0/20`) |
| `docs/architecture.md` | 14 (diagram boxes D01 to D13) |
| `docs/sack-and-trip.md` | 7.2 (lookup front door on CloudFront with AWS WAF) |
| `docs/decision-log.md` | The IDs cited on the diagram: ASM-079, ASM-082, ASM-140, PD-092, PD-095 |

### Diagram Boxes Shown

The badges follow the catalog in `architecture.md` section 14 and ASM-200:

- `[D05]` on the hub: the association of the environment with the pattern center, that is the transit gateway attachment and the Site-to-Site VPN with its customer gateway.
- `[D04]` on the production VPC and its tiers.
- `[D07]` on the transit gateway route table `centers`, whose routes and blackholes (CL-LNK-01, CL-LNK-02) are part of the cloud rule set. The perimeter firewall rules of section 4 are device configuration, not Terraform (PD-122), so the firewall carries no box badge.

### Files

- [collection-center.drawio](collection-center.drawio): Draw.io XML source with official AWS shapes.
- [collection-center.svg](collection-center.svg): vector publication export with the diagram embedded.
- [collection-center.png](collection-center.png): 2400-pixel preview export.
