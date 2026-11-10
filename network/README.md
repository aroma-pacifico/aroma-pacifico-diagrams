# Network Architecture Diagrams

This directory contains the architectural and sequence diagrams for end-to-end network connectivity, offline resilience, and latency budgets for the Aroma Pacífico coffee traceability platform (2026 update).

---

## 1. End-to-End Connectivity Architecture (`connectivity.drawio`)

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

## 2. No-Signal Handling and Crop Alert Sequence (`no-signal-sequence.drawio`)

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
