# Flows Diagrams

This folder contains the architectural, state-machine, and operational sequence diagrams for the Aroma Pacífico coffee traceability platform.

---

## Sack State Machine and Lifecycle

- Source diagram: [`sack-states.drawio`](sack-states.drawio)
- Vector export: [`sack-states.svg`](sack-states.svg)
- Bitmap preview: [`sack-states.png`](sack-states.png)

### What it shows

1. **The Seven Sack States and Design System Badges (Docs Sec 3.1, PD-154)**:
   - Maps the five states of Part C of the case statement with their accepted or rejected outcomes split out, aligning directly with the four Design System badges:
     - `in_preparation` (Part C: *en preparación* | Badge: `en preparación`): Sack packed at wet mill with 60 kg reference (ASM-150); scale weight recorded with one decimal; drying lot confirmed (ASM-096); 80-bit code bound from pre-printed batch (ASM-094); encrypted outbox buffering until cellular uplink (ASM-079).
     - `in_transit_to_center` (Part C: *en viaje al acopio* | Badge: `en viaje`): Transporter physical custody on open `farm_to_center` trip (ASM-159); GNSS fix logged every 5 minutes (PD-110); buffered offline across feeder road dead zones.
     - `at_center_accepted` (Part C: *en acopio (aceptado)* | Badge: `aceptado`): Physical custody accepted at collection center; quality verified; non-terminal; staged for outbound transport to port.
     - `at_center_rejected` (Part C: *en acopio (rechazado)* | Badge: `rechazado`): Terminal state in pilot (ASM-151); does not travel to port; reprocessed coffee presented as a new sack with a new code; rejected sack preserves historical audit trail.
     - `in_transit_to_port` (Part C: *en viaje al puerto* | Badge: `en viaje`): Open `center_to_port` trip; center operator assigned sack (`assigned_to_trip`); highway transit with 5-minute GNSS fixes.
     - `delivered_at_port_accepted` (Part C: *entregado en puerto (aceptado)* | Badge: `aceptado`): Terminal state; export handover at Buenaventura port accepted; maritime port receiver reference recorded (ASM-057, ASM-077).
     - `delivered_at_port_rejected` (Part C: *entregado en puerto (rechazado)* | Badge: `rechazado`): Terminal state; export delivery rejected at port; port receiver reference and rejection reason documented.

2. **Transition Rules, Events, Actors, and Guards (Docs Sec 3.2)**:
   - `T1` (`sack_registered`): Farmer (Farmer App, `field` pool, no MFA) -> `in_preparation`. Guard: Code issued to farm batch and unbound, weight > 0, drying lot matches farm.
   - `T2` (`picked_up_at_farm` / `sack_custody`): Transporter (Transporter App, `field` pool) -> `in_transit_to_center`. Guard: Open `farm_to_center` trip owned by caller, farm is trip pickup point (ASM-159).
   - `T3` (`received_at_center` with outcome `accepted`): Collection-center operator (Logistics App, `staff` pool, TOTP MFA) -> `at_center_accepted`. Guard: Destination center matches; missing pickup flagged if direct from `in_preparation`.
   - `T4` (`received_at_center` with outcome `rejected`): Collection-center operator (Logistics App, TOTP MFA) -> `at_center_rejected`. Guard: Destination center matches; rejection reason required (terminal, ASM-151).
   - `T5` (`bound_at_center` with outcome `accepted` or `rejected`): Collection-center operator (Logistics App, TOTP MFA) -> `at_center_accepted` or `at_center_rejected`. Guard: Sack arrives without code; label bound from center reserve (>= 400 codes, ASM-153, closes PD-096); measured reception weight; farm delivers to center. Works offline during WAN outages.
   - `T6` (`departed_to_port` / `sack_custody`): Transporter (Transporter App) -> `in_transit_to_port`. Guard: Open `center_to_port` trip owned by caller; origin is sack center; assigned to trip by operator (`assigned_to_trip`).
   - `T7` (`delivered_at_port` with outcome `accepted`): Transporter (Transporter App) -> `delivered_at_port_accepted`. Guard: Open `center_to_port` trip; port receiver reference recorded (ASM-057, ASM-077).
   - `T8` (`delivered_at_port` with outcome `rejected`): Transporter (Transporter App) -> `delivered_at_port_rejected`. Guard: Open trip; receiver reference and rejection reason required.

3. **Five Non-State-Changing Events (Docs Sec 3.2)**:
   - `arrived_at_center`: Transporter arrives at collection center yard and initiates handover; records `recorded_at` on phone and links trip to gate arrival.
   - `assigned_to_trip`: Center operator stages accepted sacks onto an outbound trip (`center_to_port`); prerequisite for `departed_to_port`.
   - `code_replaced`: Damaged label replaced from center reserve; retired code marked `void` in `sack_codes` table (returns 'not found' at lookup).
   - `weight_corrected`: Compensating event for scale typing error, recorded with reason by same role and scope (ASM-151).
   - `outcome_corrected`: Compensating event for reception classification error, recorded with reason by same role and scope (ASM-151).
   - Immutability Contract: Zero events are edited or deleted; every physical act is persisted immutably in `sack_event_log` (KMS `records`).

4. **Offline Reconciliation and Timestamp Rules (Docs Sec 3.3, Sec 4, ASM-152)**:
   - `recorded_at`: UTC date and time of the physical act to the millisecond taken on device clock; displayed in Colombia time (`America/Bogota`, UTC-5, ASM-007).
   - `received_at`: UTC timestamp applied by API upon durable persistence in DynamoDB.
   - Clock Skew Check: If `recorded_at > received_at + 5 min`, flagged as phone clock ahead for observability (PD-151).
   - Deterministic Fold: Current sack state recomputed by folding full event history ordered strictly by `recorded_at ASC, event_id ASC` (arrival order invariant).
   - Receiver-Authoritative Rule: Custody transition recorded by physical holder (T3, T4, T7, T8) applied immediately even if upstream pickup delayed in cellular dead zone.
   - Pending Hold: Events referencing an `issued` code not yet registered held pending until `sack_registered` drains.

5. **Store Architecture and Disaster Recovery (Docs Sec 9, 10, ASM-157, PD-153)**:
   - On-demand DynamoDB tables: `sack_event_log` (immutable log, KMS `records`), `sacks` (current state, KMS `records`), `sack_history` (fold source, KMS `records`), `trips` (KMS `records`), `transport_positions` (KMS `telemetry`, PD-110), `buyer_view` (KMS `public_view`, ASM-154).
   - RPO = 15 minutes: `sack-exporter` streams JSON Lines batches to S3 export prefix (ADR-001) every 60 seconds; CloudWatch alarm fires if iterator age > 5 minutes.
   - RTO = 4 hours: Public lookup runs serverless via CloudFront + WAF + Lambda URL + `buyer_view`; completely decoupled from application API.

### Source documents

- `aroma-pacifico-architecture/docs/sack-and-trip.md`:
  - Section 1 (Purpose and requirements)
  - Section 2 (Registering the sack at the farm)
  - Section 3 (Sack states and transitions, transition table T1-T8, offline rules)
  - Section 4 (Sack events: date, time, and responsible)
  - Section 5 (The trip)
  - Section 6 (Sack code and printed code sheet, center reserve >= 400 codes)
  - Section 8 (Peak load arithmetic)
  - Section 9 (Recovery objectives: RPO 15 min, RTO 4 h)
  - Section 10 (Stores, DynamoDB table topology, KMS keys)
- `aroma-pacifico-architecture/docs/security-and-identity.md`:
  - Section 3.2 (Cognito `staff` and `field` user pools)
  - Section 4.1 & 4.2 (Role catalog and matrix)
  - Section 7.2 (Customer-managed KMS keys: `records`, `telemetry`, `public_view`)
  - Section 8 (Sack code format and buyer view projection)
- `aroma-pacifico-architecture/docs/architecture.md`:
  - Section 14 (Diagram boxes D01-D13 catalog and traceability to IaC)
- `aroma-pacifico-architecture/docs/decision-log.md`:
  - `ASM-007` (Display in `America/Bogota` time)
  - `ASM-057`, `ASM-077` (Port delivery recorded by transporter with receiver reference)
  - `ASM-076`, `ASM-079` (Encrypted outbox on device, reconnect drain)
  - `ASM-091`, `ASM-092` (Passwordless SMS for field users, fresh auth for operator)
  - `ASM-094` (80-bit random code in Crockford Base32)
  - `ASM-096` (Buyer view fields)
  - `ASM-150` to `ASM-159` (Weight tolerance, terminal rejection, offline fold, center reserve, `public_view` key, recovery targets, trip legs)
  - `PD-051` (CloudFront + WAF + Lambda URL front door)
  - `PD-096` (Center code reserve for WAN outage)
  - `PD-102` (Code format and rate limit)
  - `PD-110` (DynamoDB position store)
  - `PD-151`, `PD-153`, `PD-154` (Observability hooks, rehearsal, design system badges)

### Boxes D01-D13 shown

- `D04`: Private application and data tiers in VPC (application API, Lambda `sack-events`, `sack-applier`, `sack-exporter`).
- `D05`: Collection center association (Cauca, Huila, Tolima receiving posts, local printer).
- `D06`: Identities (Cognito `staff` and `field` user pools), IAM execution roles, and customer-managed KMS keys (`records`, `telemetry`, `public_view`).

---

## Sack, Trip and Buyer Public Lookup Sequence

- Source diagram: [`sack-and-trip-sequence.drawio`](sack-and-trip-sequence.drawio)
- Vector export: [`sack-and-trip-sequence.svg`](sack-and-trip-sequence.svg)
- Bitmap preview: [`sack-and-trip-sequence.png`](sack-and-trip-sequence.png)

### What it shows

1. **Phase 1: Pre-Printed Code Sheet Generation & Printing at Center (Docs Sec 6)**:
   - Collection-center operator (TOTP MFA session) requests code sheet batch for farm allocation or center reserve.
   - Application API generates 80-bit cryptographically secure random codes (16 Crockford Base32 symbols + modulo 37 check symbol, ASM-094).
   - Batch persisted to `sack_codes` table (state: `issued`) under KMS `records`.
   - Logistics App transmits raw print job to local code-sheet printer via IPP over TCP 631 (A11 sec 5, ASM-124).
   - Sheet format: margin carries batch ID, allocation, print timestamp, and operator; labels carry only QR code (`https://<lookup-host>/s/XXXXXXXXXXXXXXXXC`), human code (`XXXX-XXXX-XXXX-XXXX-C`), and "Consulta del saco" (zero personal data, farm name, or weight).
   - Pre-printed label sheets physically distributed to wet mill.

2. **Phase 2: Sack Registration at Farm Wet Mill (Docs Sec 2)**:
   - Farmer packs 60 kg coffee sack at wet mill; scale measures weight with 1 decimal (reference 60 kg, flagged if outside Federation tolerance, never refused, ASM-150).
   - Farmer scans pre-printed label QR with Farmer App (reads code without cellular data, A09 sec 8.1).
   - Confirms drying lot ID (links sack to drying dates, coffee moisture, and grain temperature, ASM-096).
   - App records `sack_registered` event with UUID `event_id`, device `recorded_at` (UTC ms), `weight_kg`, `drying_lot_id`, and stores in encrypted local outbox (ASM-079).
   - Cellular uplink available: App uploads event over TLS 1.2+ to Application API (Cognito `field` token).
   - API verifies caller and code status, executes conditional put into `sack_event_log` (KMS `records`, immutable).
   - API returns 201 Created; app releases event from outbox.
   - DynamoDB Stream triggers `sack-applier` Lambda: creates `sacks` item (state: `in_preparation`), writes `sack_history`, and updates `buyer_view` projection (KMS `public_view`).
   - `sack-exporter` Lambda streams JSON Lines batch to S3 export prefix (ADR-001, RPO <= 15 min).

3. **Phase 3: Transporter Custody Handover & Feeder Road Trip (Docs Sec 3.2, Sec 5, A10)**:
   - Transporter opens `farm_to_center` trip on Transporter App (Cognito `field` pool).
   - Transporter scans sack QR at farm: records `picked_up_at_farm` (`sack_custody`) with device `recorded_at` and GNSS coordinates.
   - Outbox buffering: GNSS positions logged every 5 minutes while trip is open. Buffered locally across feeder road dead zones (ASM-076).
   - Cellular reconnect: Outbox drains custody events first, then GNSS positions oldest first.
   - API writes 5-minute fixes to `transport_positions` table (PK `trip_id`, SK `sequence_no`, KMS `telemetry`, PD-110).
   - API writes `picked_up_at_farm` to `sack_event_log`; Stream triggers `sack-applier`: updates `sacks` and `buyer_view` to `in_transit_to_center`.

4. **Phase 4: Collection Center Reception & Code-Sheet Binding for Unlabeled Sacks (Docs Sec 3.2, 6.3)**:
   - Transporter arrives at collection center yard: records `arrived_at_center`.
   - *Case A (Normal Labeled Sack)*: Operator scans QR -> records `received_at_center` with outcome `accepted` or `rejected` (with reason).
   - *Case B (Sack Arrived Without Code, Sec 6.3)*: Operator binds label from center reserve (>= 400 codes, ASM-153, closes PD-096), records `bound_at_center` with origin farm and measured reception weight.
   - API persists event in `sack_event_log`; Stream invokes `sack-applier`.
   - Deterministic fold: sets sack state to `at_center_accepted` (or `at_center_rejected`, terminal, ASM-151).
   - Receiver-Authoritative Rule (ASM-152): Reception applied even if farm pickup delayed in transit buffer; missing step flagged.
   - Updates `buyer_view` projection: status set to `at_center_accepted` (or `at_center_rejected`).
   - Operator stages accepted sacks onto outbound trip: records `assigned_to_trip`.

5. **Phase 5: Highway Departure & Buenaventura Maritime Port Handover (Docs Sec 3.2, Sec 5)**:
   - Transporter opens `center_to_port` trip: records `departed_to_port` (`sack_custody`).
   - `sack-applier` updates `sacks` and `buyer_view` to `in_transit_to_port`.
   - Transporter delivers at Buenaventura port: records `delivered_at_port` with outcome `accepted` and maritime receiver reference (ASM-057, ASM-077).
   - `sack-applier` folds history: transitions sack state to `delivered_at_port_accepted` (terminal state) in `sacks` and `buyer_view`.

6. **Phase 6: Buyer Public Lookup by Sack Code (Docs Sec 7, 8, ASM-058, ASM-096)**:
   - Buyer phone / browser scans QR or queries `GET https://<lookup-host>/s/XXXXXXXXXXXXXXXXC` (unauthenticated, ASM-058).
   - Amazon CloudFront edge terminates TLS 1.2+ in `us-east-1`; AWS WAF evaluates web ACL (CL-PUB-01): IP reputation, bad inputs, rate-based rule (100 req / 5 min per source IP, ASM-155), allowed path regex.
   - CloudFront routes to Lookup Lambda Function URL via Origin Access Control (OAC, `AWS_IAM` authentication, reserved concurrency 10, ASM-155).
   - Lookup Lambda validates Crockford Base32 syntax and check symbol modulo 37 (rejects 36/37 random guesses without touching DynamoDB).
   - Lookup Lambda calls `dynamodb:GetItem` on `buyer_view` table by PK `sack_code` (least privilege, KMS `public_view`, ASM-154).
   - Writes application audit record to `/aws/application/audit` CloudWatch Log Group (`anonymous`, client IP, code, timestamp, result, ASM-069).
   - Returns server-rendered HTML page (`Cache-Control: no-store`, `X-Robots-Tag: noindex`, `Referrer-Policy: no-referrer`).
   - Displays the **Four Authorized Data Groups** (ASM-096, Sec 8.2):
     1. Origin: `sack_code`, `department`, `municipality`.
     2. Processing Dates: `drying_start_date`, `drying_end_date`, `packing_date` (calendar dates, no time of day).
     3. Status: current Part C state only (`in_preparation`, `in_transit_to_center`, `at_center_accepted`/`rejected`, `in_transit_to_port`, `delivered_at_port_accepted`/`rejected`).
     4. Drying Variables: `final_coffee_moisture_pct`, `grain_temperature_min_c`, `grain_temperature_mean_c`, `grain_temperature_max_c`.

7. **Phase 7: Security & Privacy Boundary (Docs Sec 8.3, Point 2)**:
   - Explicitly withholds all sensitive data:
     - NO Personal Data: Farmer name, cédula (national ID), phone, or `subject_id` (Point 2 strictly forbids exposing cédula).
     - NO Geographic Trace: Farm name, farm ID, vereda, polygon coordinates, or plot boundaries.
     - NO Telemetry Series: Farm IoT node IDs, device X.509 certificates, or raw sensor series.
     - NO Custody Trace: Transporter identity, vehicle plate, GPS transit points, event timestamps, receiver references, or receiving operator IDs.
     - NO Internal Data: Collection center name, rejection reasons, sack scale weights, or internal `sack_id`.
     - Uniform Error Response: Malformed check symbols, void codes, unknown codes, and unregistered codes return identical 'not found' status (zero enumeration oracle).

### Source documents

- `aroma-pacifico-architecture/docs/sack-and-trip.md`:
  - Section 2 (Registering the sack at the farm)
  - Section 3 (Sack states and transitions, transition table)
  - Section 4 (Sack events: date, time, and responsible)
  - Section 5 (The trip, 5-minute GNSS fixes, outbox drain)
  - Section 6 (Sack code and printed code sheet, center reserve >= 400 codes)
  - Section 7 (Public lookup, CloudFront + WAF + Lambda URL, no load balancer, anti-enumeration, caching)
  - Section 8 (Peak load arithmetic: 30 reads/min, 0.5 reads/s)
  - Section 9 (Recovery objectives: RPO 15 min, RTO 4 h)
  - Section 10 (Stores: `sack_event_log`, `sacks`, `sack_history`, `trips`, `transport_positions`, `buyer_view`)
- `aroma-pacifico-architecture/docs/security-and-identity.md`:
  - Section 3.2 (Cognito `staff` and `field` user pools)
  - Section 4.2 (Matrix of role, data object, and action)
  - Section 7.2 (Customer-managed KMS keys: `records`, `telemetry`, `public_view`)
  - Section 8.1 & 8.2 (Sack code format and buyer view authorized fields)
  - Section 8.3 & 8.4 (Fields never returned and projection enforcement)
  - Section 9.1 (Personal data inventory and pseudonymous `subject_id`)
- `aroma-pacifico-architecture/docs/connectivity.md`:
  - Section 5 (Farm events upload)
  - Section 7 (Transporter custody and position events, outbox release on ack)
  - Section 10 (Port delivery recording)
- `aroma-pacifico-architecture/docs/landing-zone.md`:
  - Section 7.2 (Audit of a sack read, `anonymous` public reads)
- `aroma-pacifico-architecture/docs/firewall-policy.md`:
  - Rules `CL-PUB-01` to `CL-PUB-04` (WAF, CloudFront, empty `public_query` subnets)
- `aroma-pacifico-architecture/docs/architecture.md`:
  - Section 14 (Diagram boxes D01-D13 catalog and traceability to IaC)
- `aroma-pacifico-architecture/docs/decision-log.md`:
  - `ASM-057`, `ASM-077` (Transporter records port delivery with receiver reference)
  - `ASM-058` (Unauthenticated public sack lookup)
  - `ASM-069` (Audit record of anonymous sack reads)
  - `ASM-076`, `ASM-079` (Encrypted outboxes, reconnect drain priority)
  - `ASM-094`, `ASM-096` (Crockford Base32 format, buyer view projection fields)
  - `ASM-150` to `ASM-159` (Tolerance, terminal rejection, offline fold, center reserve, `public_view` key, WAF rate limit, recovery targets, trip legs)
  - `PD-051` (CloudFront + WAF + Lambda URL front door, closes PD-051)
  - `PD-096` (Center code reserve for offline WAN outages, closes PD-096)
  - `PD-102` (Code format, projection, rate limit, closes PD-102)
  - `PD-110` (DynamoDB position store, closes PD-110)
  - `PD-151`, `PD-153`, `PD-154` (Alarms, recovery rehearsal, design system badges)

### Boxes D01-D13 shown

- `D04`: VPC tiers, private subnets (application and data tiers), and empty `public_query` tier subnets.
- `D06`: Identities (Cognito `staff` and `field` user pools), IAM execution roles, and customer-managed KMS keys (`records`, `telemetry`, `public_view`).
- `D07`: AWS WAF web ACL (`CL-PUB-01`), CloudFront Origin Access Control, and security group controls.
