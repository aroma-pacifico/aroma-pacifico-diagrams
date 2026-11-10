# Network Diagrams

This directory contains network architecture diagrams for the Aroma Pacífico coffee traceability platform.

## Collection Center Network Pattern (`collection-center.drawio`)

### What It Shows

The collection center network pattern diagram depicts the parameterized LAN and WAN architecture (`10.4x.0.0/16` within `10.40.0.0/13`) deployed across the three regional collection centers in Cauca (`10.41.0.0/16`), Huila (`10.42.0.0/16`), and Tolima (`10.43.0.0/16`).

The architecture establishes:
- A standardized site boundary with seven zones (five wireless client zones, one wired infrastructure zone, and one edge transit zone).
- Perimeter Firewall Appliance [D07] providing stateful zone-based inspection (rules CC-01 to CC-DEF), dual IPsec VPN tunnel termination to AWS Hub [D05], local RADIUS EAP-TLS authentication with cached revocation list for offline network admission, DHCP and DNS conditional forwarding, and five-class traffic shaping.
- Redundant WAN uplinks connecting to the AWS Network Account Transit Gateway hub (`tgw-hub`) [D05] over a primary 100 Mbps symmetric fiber link (`10.4x.20.2`) and an automated cellular LTE+ backup link (`10.4x.20.3`).
- Private access to AWS Workload Account Production VPC (pdn `10.82.0.0/16`) tiers [D04] via Transit Gateway attachment: application tier (`10.82.48.0/20`), object storage S3 endpoint (`10.82.80.0/20`), administration tier (`10.82.96.0/20`), and explicit isolation against the reserved drone tier (`10.82.32.0/20`).
- Dedicated reception workstations in Logística with an encrypted local outbox (ASM-079) allowing coffee sack intake during WAN outages, and two Code-Sheet Network Printers in Servidores locales wired with MAC port security (ASM-124) operating strictly over LAN.
- Dedicated local internet breakout for non-operational traffic, transporter app telemetry front door, and AWS IoT Core MQTT topics (`CL-TEL-01`, `CC-DRN-04`).

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

### Inter-Zone Policy Intent Matrix

The table below summarizes the perimeter firewall policy rules (firewall-policy.md section 4) enforced statefully across zones and external destinations:

| Rule ID | Source Zone | Destination Zone | Port / Protocol | Action | Description and Justification |
|---|---|---|---|---|---|
| `CC-GST-01` | Visitantes (105) | Logística (103), Gerencia (101) | Any | Deny | Case statement point 3: guest VLAN never routes to logistics or management. Internet-only routing context. |
| `CC-GST-02` | Transporte (104) | Logística (103), Gerencia (101) | Any | Deny | Transporters are non-staff; denied to user LAN and management zones. |
| `CC-GST-03` | Visitantes (105), Transporte (104) | Private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) | Any | Deny | Isolated from all internal subnets, other centers, cloud tiers, and VPN tunnels. |
| `CC-GST-04` | Visitantes (105), Transporte (104) | Internet | TCP 25 (SMTP) | Deny | Prevents guest phones from relaying outbound spam through center breakout. |
| `CC-GST-05` | Visitantes (105), Transporte (104) | Internet | Web (TCP 80/443, UDP 443) | Allow | Non-operational internet breakout, capped per zone and client (2 / 0.5 Mbps). |
| `CC-BAS-01` | Client Zones (101-105) | Gateway (`10.4x.Z.1`) | UDP 67 (DHCP), UDP/TCP 53 (DNS), UDP 123 (NTP) | Allow | Base gateway services served locally by firewall subinterfaces. |
| `CC-BAS-02` | Switches / APs (110) | Firewall (`10.4x.10.1`) | UDP 1812, 1813 (RADIUS) | Allow | 802.1X EAP-TLS authentication requests validated against local RADIUS on firewall. |
| `CC-BAS-03` | Switches / APs (110) | Firewall (`10.4x.10.1`) | UDP 514 (Syslog), UDP 162 (SNMP traps) | Allow | Local infrastructure logs and traps exported to firewall for aggregation. |
| `CC-BAS-04` | Firewall (`10.4x.10.1`) | Switches / APs (110) | TCP 22, TCP 443, UDP 161 (SNMPv3) | Allow | Firewall management and polling of LAN switches and wireless APs. |
| `CC-LNK-01` | Primary WAN (`10.4x.20.2`) | AWS TGW VPN Peers | UDP 500, UDP 4500 (NAT-T) | Allow | Dual IPsec tunnels over primary 100M symmetric fiber link. |
| `CC-LNK-02` | Primary WAN (`10.4x.20.2`) | AWS TGW VPN Peers | IP 50 (ESP) | Allow | IPsec ESP fallback if public address is provided without NAT. |
| `CC-LNK-03` | Cellular Backup (`10.4x.20.3`) | AWS TGW VPN Peers | UDP 500, UDP 4500 (NAT-T) | Allow | Secondary IPsec tunnels over LTE+ backup router. |
| `CC-LNK-04` | Firewall Edge (`10.4x.20.1`) | AWS Route 53 Resolver (`10.82.96.0/20`) | UDP/TCP 53 | Allow | Conditional DNS forwarding of platform endpoint domain names to private cloud IPs. |
| `CC-LNK-05` | Firewall Edge (`10.4x.20.1`) | Provider DNS & NTP servers | UDP 123, UDP/TCP 53 | Allow | Firewall system time synchronization and external resolution for breakout zones. |
| `CC-LNK-06` | Logística (`10.4x.3.0/24`) | QA Application (`10.81.48.0/20`) | TCP 443 | Allow | Temporary acceptance testing of collection app against QA environment. |
| `CC-LNK-07` | Cloud Tunnel | Center LAN (except mgmt) | Any | Deny | Cloud never initiates flows into collection center, except administration session host. |
| `CC-LNK-08` | Internet (WAN interfaces) | Center LAN & Firewall | Any | Deny | All unsolicited inbound traffic dropped and logged; no WAN management. |
| `CC-LNK-09` | Center LAN | Other Centers (`10.40.0.0/13`) | Any | Deny | No center-to-center path; TGW hub enforces no spoke-to-spoke routing. |
| `CC-ADM-01` | AWS Administration (`10.82.96.0/20`) | Local Servers (`10.4x.10.0/24`) | TCP 22 (SSH), TCP 443 (HTTPS) | Allow | Session-based administration from cloud session hosts through tunnel. |
| `CC-ADM-02` | Firewall Edge (`10.4x.20.1`) | AWS Administration (`10.82.96.0/20`) | TCP 443 | Allow | CloudWatch Logs/Metrics and IAM Roles Anywhere interface endpoints. |
| `CC-ADM-03` | Client Zones (101-105) | Local Servers (`10.4x.10.0/24`) | Any | Deny | No user zone can reach management interfaces of network devices or firewall. |
| `CC-ADM-04` | Printers (`10.4x.10.50-.59`) | Any | Any | Deny | Printers never initiate outbound connections to LAN, cloud, or internet. |
| `CC-ADM-05` | Local Servers (`10.4x.10.0/24`) | Vendor Update Services | TCP 443 | Allow | Approved maintenance window firmware and security signature updates. |
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
| `CC-TEL-01` | Center LAN | AWS Telemetry Ingress (`10.82.16.0/20`) | Any | Deny | Farm telemetry never crosses a collection center; routed directly to IoT Core. |
| `CC-TEL-02` | Transporte (104) | Public API Front Door | TCP 443 | Allow | Transporter location updates and custody events via breakout (CloudFront/WAF). |
| `CC-PUB-01` | Client Zones (101, 102, 104, 105) | Public Sack Lookup | TCP 443 | Allow | Public sack verification front door reached via local breakout. |
| `CC-DEF-01` | Any | Any | Any | Deny | Explicit default deny; all unmatched flows dropped and logged for auditing. |

### Source Documents and Sections

- `aroma-pacifico-architecture/docs/case/case-statement.pdf`
- `aroma-pacifico-architecture/docs/collection-center-network.md` (Section 2: One pattern, three centers; Section 3: Zones, VLANs and SSIDs; Section 4: Where each design element connects; Section 5: Logistics workstations and code-sheet printing; Section 6: Inter-zone policy intent; Section 7: Capacity; Section 8: Non-operational traffic control; Section 9: Hardware device classes; Section 10: Behavior when the WAN fails)
- `aroma-pacifico-architecture/docs/firewall-policy.md` (Section 4: Perimeter firewall of each collection center, Rules CC-01 to CC-DEF; Section 6: Rules to audit before production, and temporary rules; Section 7: Traceability to the A11 intent matrix)
- `aroma-pacifico-architecture/docs/addressing-plan.md` (Section 2: Reserved collection-center space `10.40.0.0/13`; Section 3: Top-level allocation; Section 4: Subnet layout inside each environment VPC)
- `aroma-pacifico-architecture/docs/architecture.md` (Section 14: Diagram boxes D01 to D13 and traceability to the IaC)
- `aroma-pacifico-architecture/docs/decision-log.md` (Assumptions ASM-080 to ASM-089, Decisions PD-090 to PD-096, ASM-079, ASM-124, ASM-125, ASM-140, ASM-200)

### Diagram and Box Catalog Traceability

The diagram models the following architecture boxes cataloged in `architecture.md` section 14:
- `[D05]` Association of Environment with Pattern Center (implemented by IaC box B05: AWS Transit Gateway hub `tgw-hub`, dual Site-to-Site VPN connection, static routing for `10.4x.0.0/16`, certificate customer gateway).
- `[D07]` Pattern Firewall Rules (implemented by IaC box B07: Perimeter firewall rule set CC-01 to CC-DEF, stateful inspection, zone subinterfaces, RADIUS EAP-TLS service, and traffic shaper).
- `[D04]` VPC Tiers of Pattern Environment (implemented by IaC box B04: Production VPC pdn `10.82.0.0/16` tiers: transit, application, object_storage, administration, and reserved drone_object_ingress tier).

### Files

- [collection-center.drawio](collection-center.drawio): Source diagram in Draw.io XML format with official AWS vendor icons.
- [collection-center.svg](collection-center.svg): Headless vector publication export with embedded diagram XML.
- [collection-center.png](collection-center.png): High-resolution preview export (~2400px wide, visual self-inspection verified).
