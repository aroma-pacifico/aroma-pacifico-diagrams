# A13 Data Paths

This document traces the crop and drying telemetry, transporter position, and
drone object flows from field input to persistence and their primary
consumers. It follows the architecture decisions available on 2026-11-04.
The security and identity source was read from
`origin/docs/9-security-and-identity-main` at revision
`ec70b3a4265a3eca7869425c035126bf3178ce53`; it had not merged to the
architecture repository's `origin/main` at review time. The other cited
architecture sources were read from `origin/main` at revision
`6375c06816b0e05b0708dfab4e0429d39c2ec4f5`.

## Crop and drying telemetry

Each farm has a crop node and a drying node. The crop node reports the case
statement's six daily readings; an active drying node reports 24 readings per
day. Both send every reading, including its original timestamp, rather than
evaluating thresholds on the device. The IoT topic rule writes each accepted
reading and invokes the threshold function as separate actions; the source
architecture does not specify their ordering or failure coupling (PD-112).
When the route is available, the case
statement's alert target is under five minutes; after a signal outage, buffered
readings are evaluated as they drain, within the 24-hour buffer limit.

| Field | Data path |
|---|---|
| Input | Crop variables (nitrogen, phosphorus, potassium, soil and air moisture and temperature) every four hours; drying coffee moisture and temperature hourly while a lot is drying. Each reading includes node identity, original timestamp, sequence number and values. |
| Protocol | MQTT 3.1.1 or 5 over TLS 1.2 or later, mutual X.509 authentication, QoS 1; port 8883, with MQTT over TLS on port 443 using ALPN when 8883 is blocked. Farm traffic goes by cellular bearer and public internet to the regional AWS IoT Core endpoint in `us-east-1`; it does not pass through a collection center or VPC subnet. |
| Buffer | Encrypted non-volatile node buffer, at most 24 hours. New live readings are sent first after reconnect; remaining records drain oldest first. A record is released only after MQTT PUBACK. An expired unacknowledged record is evicted and counted; the next message reports the gap. |
| Persistence and retention | An AWS IoT Core topic rule writes readings to the DynamoDB telemetry table, keyed by node identifier and original reading timestamp for idempotent drain. Accepted readings are retained for five years. |
| Event emitted | Each accepted publication is the A08 telemetry-publication audit record. The IoT rule also invokes the threshold function; when a reading is outside the Federation-configured range, the function persists `crop_alert_raised` and emits the farmer alert through the SNS alert topic. The five-minute alert budget is a design target, not a measured result. |
| Main consumer | The threshold function evaluates configured ranges. The farmer history and pilot report read the persisted telemetry history; out-of-range results feed the crop-alert case and farmer notification. |
| Encryption | MQTT TLS 1.2 or later in transit; encrypted node buffer at rest; DynamoDB at rest under the `telemetry` customer-managed KMS key. |
| A08 audit | Accepted readings and their certificate principal, client identifier, source IP, receive time and original timestamp are recorded by the IoT topic rule in the telemetry store. IoT Core `ERROR` logging in `AWSIotLogsV2` records rejected connection and publication attempts. |

The reserved `telemetry_ingress` VPC tier hosts private interface endpoints for
private paths, such as collection-center connectivity. It is not on the farm
node path; the public IoT Core endpoint is the farm ingress. The diagram also
marks the collection-center boundary as not traversed by these readings.

## Transporter position and offline events

The transporter phone records a GNSS fix every five minutes while a trip is
open. A position is persisted to the encrypted outbox even when the phone has
no data connection. Custody changes use the same outbox and are sent ahead of
positions on reconnect, preserving the event time and trip order.

| Field | Data path |
|---|---|
| Input | GNSS position every five minutes while in transit, plus custody events such as pickup, arrival at a center, departure to port and port delivery. Events include a client-generated `event_id`, `trip_id`, transporter identity, device ID, original `recorded_at`, sequence number and link state. |
| Protocol | HTTPS over TLS 1.2 or later from the phone to the public application API, using the transporter identity. At a collection-center yard, the phone may use the transport SSID; its traffic still exits through the public application endpoint and does not enter the center-to-cloud VPN. |
| Buffer | Encrypted phone outbox survives app restarts and reboots. It has no time-based eviction and releases an event only after API acknowledgement. It is sized for at least seven days of five-minute positions (2,016 fixes) plus trip custody events. If position capacity is exhausted, positions are thinned to one per 15 minutes before any are dropped; custody events are never evicted. |
| Persistence and retention | The API persists `transport_position` and `sack_custody` events in the position/transactional store allowed by Part E. The source architecture does not choose a concrete service for the position store. Event records and positions are retained for five years, then deleted by the configured retention process. |
| Event emitted | The app persists `transport_position` for each GNSS fix and `sack_custody` for each custody act before upload. On reconnect, custody events drain first, followed by positions oldest first. The API deduplicates on `event_id` and reconstructs order from `recorded_at` and `sequence_no`, not arrival order. |
| Main consumer | The trip trace used by the transporter and collection-center operations while the trip is open; custody events also update the sack history. |
| Encryption | HTTPS TLS 1.2 or later in transit; phone outbox encrypted at rest with a key in the phone keystore; persisted position data encrypted at rest under the `telemetry` KMS key, and custody records under the `records` key. |
| A08 audit | A08's four mandatory audit events do not include position or custody publication. These business events remain in their persisted event records; this diagram does not add a CloudTrail data-event audit for them. |

If the phone loses data, the event remains locally persisted and its
`link_state=offline` and original timestamp identify the gap. When connectivity
returns, custody drains first and positions drain oldest first. If position
capacity is exhausted, the documented thinning rule preserves the trail at a
15-minute interval before dropping any position. The collection-center yard
path uses the 10.4x.4.0/24 transport zone and local internet breakout; the
private VPN carries logistics and drone traffic, not transporter events.

## Drone objects for confirmation, verification and estimation

The contracted operator receives missions preloaded at a municipal seat or a
collection center. Confirmation objects target a sensor-marked focus;
verification objects close a post-fumigation case only after persistence;
estimation objects carry an area flight's harvest estimate. The same object
ingress also holds maps, polygons and flight records. Object persistence is
the prerequisite for downstream inference and for closing a fumigation.

| Field | Data path |
|---|---|
| Input | Photos and other objects for focus confirmation, post-fumigation verification and harvest estimation, along with polygons, maps and flight records. Mission package (order, polygon and no-fly zones) is downloaded before flight. |
| Protocol | HTTPS with TLS 1.2 or later. The ground station uploads to Amazon S3 using short-lived pre-signed URLs for the order's prefix and multipart checksums. From a center, the path is Support (VLAN 102, 10.4x.2.10), center firewall, IPsec site-to-site VPN to the network-account hub, the workload VPC `transit` tier, then interface endpoints in `drone_object_ingress` and `object_storage`. From a municipal seat it uses the public S3 endpoint in `us-east-1`. |
| Buffer | Drone and ground-station storage has no specified time limit. An interrupted upload resumes by multipart part; an object is removed from ground-station storage only after the platform confirms persistence with a matching checksum. If signal is lost at the farm, the loaded mission proceeds and its record and objects remain buffered until connectivity returns. |
| Persistence and retention | Amazon S3 drone-object bucket in the workload account, with object prefixes scoped to the order. Photos, maps, polygons and flight records are retained for five years. The persisted object is linked to its flight record by the reception function. |
| Event emitted | A08 audits delivery using CloudTrail S3 data events `PutObject` and `CompleteMultipartUpload`; the reception function links the object to its flight record. The specific inference trigger, retry policy and dead-letter/error observation path remain for A17 to decide (PD-111). |
| Main consumer | The confirmation, verification and estimation inference functions read the persisted object and write a timestamped result, confidence and source-object identifier to the case/flight record. Confirmation does not authorize fumigation; verification cannot close a case until its result is persisted; estimation does not create a packing order. |
| Encryption | HTTPS TLS 1.2 or later in transit, with bucket policy denying insecure transport; encrypted drone and ground-station buffers; S3 SSE-KMS under the `objects` customer-managed key at rest. Objects containing a person are not sent to inference; their handling remains open in PD-015. |
| A08 audit | CloudTrail records S3 `PutObject` and `CompleteMultipartUpload` data events for the production drone bucket, including bucket, key, version, size, principal, time, source IP and user agent. The application associates the delivery with the order and flight record. The individual pilot identity in the audit is still open in PD-075. |

When the link is down, the dashed diagram branch records the local buffer and
the fact that the order remains open. A flight record must be persisted before
the fumigation reaches `executed`; a verification result must also persist
before it reaches `closed`. From the collection center the ground station is
placed in Support and uses private endpoints; the municipal-seat route uses
public endpoints and does not cross a VPC tier.

## Assumptions and pending decisions for the decision log

No new assumptions are introduced here. The source architecture already
defines the intervals, protocols, buffer behavior, retention, AWS stores and
encryption classes used above. These open points are recorded for the lead to
copy into the architecture decision log:

| ID | Status | Decision needed |
|---|---|---|
| PD-110 | Open | Choose the concrete service and keying/idempotency design for transporter positions. The architecture specifies the allowed transactional or time-series store class but does not select a service. |
| PD-111 | Open | A17 must select and document the persisted-object inference trigger, retry and dead-letter behavior, and failure observation point. The current source documents the inference behavior but not these mechanisms (PD-014). |
| PD-112 | Open | Define the ordering and failure coupling between the IoT topic rule's telemetry-store write and threshold-function invocation. The current source specifies both actions but does not state whether one waits for the other. |

No ASM-110 to ASM-119 assumption is needed for this task. The architecture
security source also keeps PD-015 open for objects containing people and
PD-075 open for carrying the individual pilot into the drone upload audit.

## Architecture references

| Source | Sections used |
|---|---|
| `aroma-pacifico-architecture/docs/case/case-statement.pdf` | Crop/drying cadence, five-year retention, alert latency and offline alert behavior, transporter interval, drone workflow, inference requirements. |
| `aroma-pacifico-architecture/docs/adr/adr-001-primary-and-second-cloud.md` | AWS primary Region `us-east-1`, account and environment model. |
| `aroma-pacifico-architecture/docs/adr/adr-002-drone-flight-mode-and-governance.md` | Combined flight mode, service events, mission preload, object buffer and closure rule. |
| `aroma-pacifico-architecture/docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` | IoT Core ingestion, DynamoDB telemetry store, threshold function and alert flow. |
| `aroma-pacifico-architecture/docs/addressing-plan.md` section 4 | VPC tiers and private interface endpoints. |
| `aroma-pacifico-architecture/docs/connectivity.md` sections 3, 4, 6, 7, 8 and 9 | Protocols, buffers, no-signal behavior, transporter events, alert budget, center WAN and drone endpoints. |
| `aroma-pacifico-architecture/docs/collection-center-network.md` sections 3, 4.1 and 10 | Support/VLAN 102, drone ground-station placement and WAN behavior. |
| `aroma-pacifico-architecture/docs/landing-zone.md` section 7 | Audit of telemetry publication and drone object delivery. |
| `aroma-pacifico-architecture/docs/security-and-identity.md` sections 6, 7, 9 and 11, read from `origin/docs/9-security-and-identity-main` | Device identities, encryption per store, retention and audit inputs. This source had not merged to architecture `origin/main` when inspected. |
| `aroma-pacifico-architecture/docs/decision-log.md` | PD-014, PD-015, PD-073 and PD-075. |

## Diagram verification record

The diagrams are `telemetry.drawio`, `transporter-position.drawio` and
`drone-objects.drawio`. They are uncompressed, one-page Draw.io diagrams and
use AWS `mxgraph.aws4.*` service shapes. Each arrow identifies the protocol
and the local-buffer behavior where applicable. The dashed path identifies
the no-signal case. Semantic review is tied to the source table above; the
PNG exports were inspected individually at their 2400-pixel export width.
The local renderer is Draw.io CLI 32.4.1.
