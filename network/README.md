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
  - `Reserved Tiers` (`10.E.112.0/17`, eight /20 blocks): 32,768 unallocated private addresses per environment for future architecture evolution.
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
