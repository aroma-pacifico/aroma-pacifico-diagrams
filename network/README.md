# Network Diagrams

This directory contains network architecture diagrams for the Aroma Pacífico coffee traceability platform.

## Collection Center Network Pattern (`collection-center.drawio`)

### What It Shows
The collection center network pattern diagram depicts the parameterized LAN and WAN architecture (`10.4x.0.0/16` within `10.40.0.0/13`) deployed across the three regional collection centers in Cauca (`10.41.0.0/16`), Huila (`10.42.0.0/16`), and Tolima (`10.43.0.0/16`).

Key components and architectural patterns shown:
- **Site Boundary and Seven Zones**:
  - `ENLACE A LA NUBE` (VLAN 120, `10.4x.20.0/24`): Wired edge interfaces, Primary WAN (`10.4x.20.2`, 100 Mbps symmetric fiber), Cellular backup router (`10.4x.20.3`, LTE+ failover).
  - `GERENCIA` (VLAN 101, `10.4x.1.0/24`, SSID `cc-management`): WPA3-Enterprise (802.1X EAP-TLS), 7 planned clients, Staff traffic class (Priority 4, 28/7 Mbps guaranteed).
  - `SOPORTE` (VLAN 102, `10.4x.2.0/24`, SSID `cc-support`): WPA3-Enterprise, 7 staff clients plus Drone Ground Station (`10.4x.2.10/32` fixed DHCP reservation, ADR-002, ASM-082).
  - `LOGÍSTICA` (VLAN 103, `10.4x.3.0/24`, SSID `cc-logistics`): WPA3-Enterprise, 2 wired reception workstations (`10.4x.3.2` and `10.4x.3.3`) running Part C collection app with operator MFA and encrypted local outbox (ASM-079), plus 20 handheld scanners across the yard. Logistics traffic class (Priority 2, 44/11 Mbps guaranteed).
  - `TRANSPORTE` (VLAN 104, `10.4x.4.0/24`, SSID `cc-transport`): WPA3-Personal (SAE rotated per harvest), client isolation enabled, 20 planned transporter phones, Internet-only routing context.
  - `VISITANTES` (VLAN 105, `10.4x.5.0/24`, SSID `cc-guest`): WPA3-Personal (SAE weekly rotated), client isolation enabled, 10 planned phones, Internet-only routing context. Non-operational traffic class (Priority 5, capped at 10/2.5 Mbps, 2/0.5 Mbps per client, ASM-125).
  - `SERVIDORES LOCALES` (VLAN 110, `10.4x.10.0/24`): Wired only, no SSID. Network control traffic class (Priority 1, 2/2 Mbps guaranteed). Contains two Code-Sheet Network Printers (`10.4x.10.50` to `.59`) wired with MAC port security (ASM-124).
- **Core Network Appliances**:
  - Perimeter Firewall Appliance [D07]: Stateful zone-based inspection (CC-01 to CC-DEF), IPsec VPN termination (two tunnels: UDP 500/4500, ESP IP 50), subinterfaces, DHCP server, DNS conditional forwarding, local RADIUS server, and QoS traffic shaper (Priorities 1 to 5).
  - Managed PoE Access Switch and Wi-Fi APs: 802.1Q trunk (VLANs 101-105, 110), 802.1X port admission, 5 SSIDs. Layer 2 only (ASM-080).
- **External Cloud and Internet Connectivity**:
  - AWS Network Account Hub [D05] (us-east-1): Transit Gateway hub (`tgw-hub`), site-to-site VPN connection, static route table `centers` (`10.4x.0.0/16` only, CL-LNK-06), certificate-based customer gateway (PD-120).
  - AWS Workload Account Production VPC (pdn `10.82.0.0/16`) [D04]: Dedicated tiers reached via Transit Gateway attachment (transit, application, object_storage, administration, and reserved drone_object_ingress tier).
  - Public Internet and AWS Edge Services: Local breakout for non-operational traffic, transporter app telemetry front door, and AWS IoT Core MQTT topic (`CL-TEL-01`, `CC-DRN-04`).
- **Policy Intent**:
  - Allowed flows (solid green arrows) and Denied flows (dashed red arrows) reflecting the inter-zone stateful firewall matrix (Case statement points 2, 3, 4; firewall-policy.md section 4).
  - Code-sheet printers never initiate connections (CC-ADM-04).
  - No general internet for logistics or servers (CC-LOG-04, CC-ADM-07).
  - Transporters and visitors isolated from internal LAN (CC-GST-01, CC-GST-02, CC-GST-03).
  - Inbound internet traffic blocked and logged (CC-LNK-08).

### Source Documents and Sections
- `aroma-pacifico-architecture/docs/case/case-statement.pdf`
- `aroma-pacifico-architecture/docs/collection-center-network.md` (all sections)
- `aroma-pacifico-architecture/docs/firewall-policy.md` (Section 4: Collection Center Pattern Rules CC-01 to CC-DEF, Section 6: Local Breakout, Section 7: Inter-zone Matrix)
- `aroma-pacifico-architecture/docs/addressing-plan.md` (Section 2: CIDR Blocks, Section 3: Collection Center Subnets, Section 4: Cloud Subnets)
- `aroma-pacifico-architecture/docs/architecture.md` (Section 14: Diagram and Box Catalog)
- `aroma-pacifico-architecture/docs/decision-log.md` (Assumptions ASM-080 to ASM-089, Decisions PD-090 to PD-096, ASM-079, ASM-124, ASM-125, ASM-140, ASM-200)

### Diagram and Box Catalog Traceability
The diagram includes the following architecture boxes defined in `architecture.md` section 14:
- `[D05]` Association of Environment with Pattern Center (implemented by IaC box B05: TGW hub, VPN connection, static routing 10.4x.0.0/16)
- `[D07]` Pattern Firewall Rules (implemented by IaC box B07: Perimeter firewall rule set CC-01 to CC-DEF)
- `[D04]` VPC Tiers of Pattern Environment (implemented by IaC box B04: Production VPC pdn tiers: transit, application, object_storage, administration, reserved drone tier)

### Files
- [collection-center.drawio](collection-center.drawio): Diagram source in Draw.io XML format.
- [collection-center.svg](collection-center.svg): Headless vector export with embedded diagram XML.
- [collection-center.png](collection-center.png): High-resolution preview export (~2400px wide).
