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
  x = longitude, y = latitude; z optional.
- `StatusReport` [verified: status_report.proto]: report_id (ULID), system (OK/WARNING/ERROR/GOODBYE), info (NEW/UNCHANGED), mode
  mandatory; optional active_task_id, power, node_location, field_of_view, coverage, obscuration, status[] (level, type, value).
  `SYSTEM_GOODBYE` = about to disconnect.
- `Task` [verified: task.proto]: task_id (ULID), control (START/STOP/PAUSE) mandatory; name, description, start/end time, region[],
  command oneof: request (string), detection_threshold, detection_report_rate, classification_threshold (LOW/MEDIUM/HIGH),
  mode_change, look_at, move_to, patrol, follow.
- `TaskAck`: task_id, task_status (ACCEPTED/REJECTED/COMPLETED/FAILED) mandatory; reason[]. `Error`: packet (bytes of the bad message), error_message[].
- Registration `Duration` = units + float value; `status_definition.status_interval` declares the heartbeat cadence.
- Mandatory/UUID/ULID constraints are embedded as options in the protos (`proto_options.proto`) → usable for generic validation.
- Classification taxonomy: in the standard §7.2, NOT in the protos. Don't invent class names. The Dstl harness embeds a
  taxonomy (`Air Vehicle` → `UAV Rotary Wing` …) but does not enforce it on detections [verified: CommonValidator.cs]; list in `sapient-sensor-node.md`.

## Roles and flow [verified: Scout ↔ Dstl harness captures 2026-11-26]
Sensor (edge node / ASM) connects as TCP client → sends Registration → fusion node replies RegistrationAck (0.3–0.5 s) →
sensor sends periodic StatusReports (heartbeat) and DetectionReports → fusion node may send Task → sensor replies TaskAck.
The Scout sends detections without any start Task. Harness `Request.Start`/`Stop` tasks are `command.request` "Start"/"Stop"
with `control: CONTROL_START` [verified: Default.Task.Request.*.json].

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
- Keep reporting an object while it is detected, also when static; declared status_interval = real cadence;
  INFO_UNCHANGED when nothing changed; SYSTEM_GOODBYE before a planned disconnect (lessons from the Scout, `sensors/scout.md`).
- Must pass the Dstl harness first: checklist and acceptance test in `sapient-sensor-node.md`.

## Upstream connection model (decision, see decisions.md)
Option A: one SAPIENT connection per sensor node, all to the same c2lynx port.
Deferred: B (Pi as child fusion node using `dependent_nodes`); C (several node_ids on one connection — unconfirmed).

## Open questions
- Can one TCP connection carry several node_ids? (standard)
- Meaning of start/stop requests in the standard (the harness only shows their encoding, see Roles and flow).
- Stale rules in the standard (cadence is declared per node in `status_interval`; the harness SDA drops a client after 60 s silence).
- Fusion-node hierarchy rules (`dependent_nodes`, `reporting_region`).
