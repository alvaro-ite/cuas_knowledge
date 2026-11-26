# SAPIENT (BSI Flex 335 v2.0) — what we know

## Sources
- Standard: BSI Flex 335 v2.0:2024-03 (free download from BSI; copyrighted). Not fully read yet.
- Protos: github.com/dstl/SAPIENT-Proto-Files, folder `bsi_flex_335_v2_0`, pinned commit
  `244a741c13bcee628f84944aef91e4a66854bd7d`. Compile them; never hand-write message classes.
  The protos import `sapient_msg/...` → rebuild that folder layout before compiling.
- Dstl Test Harness code: github.com/dstl/BSI-Flex-335-v2-Test-Harness.

## Transport [verified: harness code SapientServices/SocketCommsCommon.cs, SendPacket]
- TCP. Each message = 4-byte unsigned little-endian length + protobuf `SapientMessage` bytes.
- Harness configs carry a legacy `sendNullTermination=True` setting but SendPacket adds no null → tolerate a stray trailing 0x00.
- The standard does not define bearer or security; TLS/auth are our choice [doc: BSI product page].

## Messages [verified: sapient_message.proto]
Envelope `SapientMessage`: `timestamp` (mandatory, UTC send time), `node_id` (mandatory, UUID of sender),
`destination_id` (optional, UUID), `content` oneof (mandatory): registration, registration_ack,
status_report, detection_report, task, task_ack, alert, alert_ack, error; `additional_information`.
- `RegistrationAck`: `acceptance` (mandatory bool), `ack_response_reason` (repeated string).
- `Registration`: node_definition, icd_version, name, short_name, capabilities, status_definition,
  mode_definition, config_data (mandatory per proto options), `dependent_nodes` (child nodes),
  `reporting_region` (child fusion node → parent).
- `DetectionReport`: report_id + object_id (mandatory ULIDs), task_id, state, location or range/bearing,
  detection_confidence (0–1), track_info / object_info (type/value/error pairs declared in Registration),
  classification (type, confidence, sub_class), behaviour, signal (RF), associated/derived detections,
  velocity, colour, `id` (object ID, e.g. tail number).
- `Location`: x, y, z + errors; coordinate system e.g. `LAT_LNG_DEG_M`; datum `WGS84_E` (ellipsoid) or `WGS84_G` (geoid).
- Mandatory/UUID/ULID constraints are embedded as options in the protos (`proto_options.proto`) → usable for generic validation.
- Classification taxonomy: in the standard §7.2, NOT in the protos. Don't invent class names.

## Roles and flow [inferred from message types; confirm in standard]
Sensor (edge node / ASM) connects as TCP client → sends Registration → fusion node replies RegistrationAck →
sensor sends periodic StatusReports (heartbeat) and DetectionReports → fusion node may send Task → sensor replies TaskAck.
The harness ships `Task.Request.Start` / `TaskAck.Request.Start` templates → some nodes may wait for a start task [inferred].

## Contract: our SAPIENT server (fusion-node side, PC agent)
- TCP server, many clients on one port; identify nodes by `node_id`, not IP.
- Per-connection session: CONNECTED → REGISTERED → ACTIVE → CLOSED. Reply RegistrationAck with
  `destination_id` = sensor's node_id. Re-registration of a known node_id replaces the old session.
- StatusReport = heartbeat; node flagged stale after a configurable timeout.
- Validates with the proto options; never drops a message for being invalid, logs it.
- Can send Task on demand (start/stop/status/registration requests) and logs TaskAck.
- Logs raw bytes per connection + decoded JSONL per message.
- Listening port / our node_id / bind address: TBD by the PC agent → record here when fixed.

## Contract: our SAPIENT sensor nodes (Scout via its native output; ADS-B node on the Pi)
- TCP client to the fusion node; reconnect with backoff; re-register after every reconnect.
- Stable `node_id` (UUID) per sensor, kept across restarts. Record it in the sensor's file.
- One `object_id` (ULID) per tracked object, stable for the track's life; real identifier (ICAO, UAS ID) goes in `id`.
- Locations with explicit coordinate system and datum; errors filled when known.
- Anything without a native field goes in `object_info`, declared in the Registration.
- Never report confidence 1.0 for self-declared data (RID, ADS-B can be spoofed).

## Upstream connection model (decision, see decisions.md)
Option A: one SAPIENT connection per sensor node, all to the same c2lynx port.
Deferred: B (Pi as child fusion node using `dependent_nodes`); C (several node_ids on one connection — unconfirmed).

## Open questions
- Can one TCP connection carry several node_ids? (standard)
- Exact Task fields for start/stop requests (task.proto + standard).
- StatusReport cadence and stale rules (standard).
- Fusion-node hierarchy rules (`dependent_nodes`, `reporting_region`).
