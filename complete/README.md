# Complete Architecture Diagram

`architecture.drawio` is the editable, uncompressed source. Page 1, **Complete architecture**, shows runtime paths, trust boundaries, D01-D13 badges on runtime and IaC components, and labelled flow arrows. Page 2, **IaC traceability**, maps Track B to boxes D01 through D13 and shows the reviewable plan pipeline. The SVG files are publication exports; the PNG files are 2400-pixel previews.

The source architecture documents and ADRs were read from `origin/main` of `aroma-pacifico-architecture` at revision `0140564f1b4d9a8378db3d4d0f59e9d1934e6c18`. Requirements were checked against `docs/case/case-statement.pdf` in that repository. Draw.io CLI 32.4.1 rendered both pages and the updated A13 diagrams. This CLI uses one-based page indexing, so page 2 was exported with `-p 2`.

| Architecture source | Diagram content grounded by the source |
|---|---|
| `docs/addressing-plan.md`, `docs/landing-zone.md` | Environment accounts, CIDRs, VPC tiers, AWS organization, security and audit boundary. |
| `docs/connectivity.md`, `docs/collection-center-network.md`, `docs/firewall-policy.md` | Farm, center, drone, transporter and port paths; center pattern, WAN, policies and permanent firewall references. |
| `docs/security-and-identity.md`, `docs/adr/adr-001-primary-and-second-cloud.md` | AWS primary Region, undeployed Azure export target, identities, MFA, encryption and data boundaries. |
| `docs/drone-service.md`, `docs/adr/adr-002-drone-flight-mode-and-governance.md` | Combined flight mode, mission preload, pest-to-fumigation events, human authorization and closure gates. |
| `docs/farm-monitoring.md`, `docs/sack-and-trip.md` | Telemetry and alert path, sack lifecycle, trip custody, and buyer view. |
| `docs/inference-functions.md`, `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md`, `docs/adr/adr-004-drone-capability-orchestration.md` | EventBridge, SQS, Lambda, Amazon Bedrock, remote state, the approved pattern module and plan pipeline; the bus, rules and per-function queues are labeled the Drone capability orchestrator (event-driven), ADR-004. |
| `docs/automation-and-pipelines.md`, `docs/legacy-2020.md`, `docs/decision-log.md` | Promotion gates, harvest-week window, retained and retired 2020 requirements, and open/closed decision status. |
| `aroma-pacifico-diagrams/data-paths/data-paths.md` and the three A13 Draw.io sources | Detailed telemetry, transporter and drone paths reconciled to the same architecture revision. |

## Traceability Legend

The D badges identify the Track B component implemented by the corresponding roadmap issue. They use the same numbers as the architecture document.

| Box | Track B component |
|---|---|
| D01 | Private infrastructure repository with one root stack per environment and tagged reusable modules from `aroma-pacifico-terraform-modules`. |
| D02 | One encrypted, versioned S3 state bucket per environment with native S3 locking (ADR-003). |
| D03 | Per-environment configuration; secrets stay outside the repository and state. |
| D04 | Pattern VPC and tiers for the `pdn` environment. |
| D05 | Association of the `pdn` environment with one pattern collection center, Cauca. |
| D06 | Base identities, MFA, KMS keys and initial access controls. |
| D07 | Permanent pattern firewall rules. |
| D08 | Reusable module for one pattern farm, not all 200 farms. |
| D09 | Approved telemetry-ingestion pattern. |
| D10 | Event-driven inference-function pattern. |
| D11 | CI pipeline that produces an executable, reviewable plan. |
| D12 | CD pipeline with `pdn` approval and the harvest-week change window. |
| D13 | Final plan validation and evidence. |

Solid arrows show ordered business or data progression. Orange dashed arrows show no-signal branches and locally buffered work. The numbered pest-to-fumigation arrows mean:

| Step | Event and gate |
|---|---|
| 1 | Persist the sensor reading through AWS IoT Core. |
| 2 | Evaluate the Federation-configured threshold and persist `crop_alert_raised`. |
| 3 | A contracted operator flies a confirmation mission to the sensor-marked focus. |
| 4 | The confirmation function calls Amazon Bedrock and persists `focus_confirmed` with timestamp, confidence and source-object ID. |
| 5 | A Federation agronomist authorizes the order as a human identity with MFA. |
| 6 | The contracted operator fumigates the authorized plot. |
| 7 | Persist the flight record, including operator, window and product-lot identifier. |
| 8 | Persist the verification object and result before `case_closed`; a missing object or failed invocation leaves the case open. |

`crop_alert_raised` enters precision mode. `focus_confirmed` makes an order eligible but does not authorize it. A persisted plot polygon and closed confirmation flight record are prerequisites to releasing the order. Area mode handles plot delimitation and harvest estimation; precision mode handles focus confirmation, fumigation and verification of the same plot.

## Architecture Notes

The runtime page includes AWS `dev`, `qa` and `pdn` environments in `us-east-1`, with the production VPC nested inside the production account boundary (`pdn`), their VPC CIDRs and tiers, the three collection-center ranges, firewall policy references, the separate public farm and transporter paths, the center VPN path, the private drone upload path via `application` and `object_storage` endpoints (with `drone_object_ingress` reserved under ASM-140 and CC-DRN-02), and the municipal-seat public S3 path. It also shows the buyer's limited sack view, five-year retention, encrypted store-and-forward, and the AWS organization and shared security/network accounts. Azure is represented as a prepared, undeployed export target under ADR-001 (`10.88.0.0/13`); the diagram adds no Azure account or cross-cloud link. All runtime flows are drawn as labelled directional arrows.

The legacy note records what the update retains from 2020 (farm telemetry, alerts and sack traceability) and retires (assumed farm coverage, direct device-to-ingestion publishing and mobile-generation dependence). Vessel tracking remains out of scope.

The current A17 design uses a persisted mission record, EventBridge Pipe and bus, a per-function SQS queue, Lambda and Amazon Bedrock. PD-111 and PD-112 are closed in the source decision log; PD-110 is closed by A16. Objects containing a person are excluded from inference under closed PD-015. Open source decisions relevant to the pipeline remain PD-019 (plan evidence under the plan-only boundary), PD-020 (implementation of the required production approval gate), and PD-167 (model-quality evaluation before production). These are not newly raised by this diagram.

## Assumptions and Pending Decisions for the Decision Log

No new assumption or pending decision was introduced by this diagram. ASM-190 through ASM-199 and PD-190 through PD-199 are unused. Existing open items retain their source IDs and are not duplicated here.
