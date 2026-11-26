# Building a SAPIENT sensor node that connects to our harness

For the agent implementing a sensor node (ADS-B on the Pi first). Protocol facts: `sapient.md`.
Harness setup and internals: `test-harness.md`. Everything below was checked against the harness code
or live captures on 2026-11-26 unless tagged otherwise.

## Where to connect
| sensor | connect to (test PC) | path inside the PC |
|---|---|---|
| Scout | TCP :14100 | SapientTap → SDA :14005 (ASM1 slot) |
| ADS-B node | TCP :14101 | SapientTap (`Start Tap ADS-B.bat`) → SDA :14006 (ASM2 slot, `Start ADSB DataAgent.bat`) |
| next sensor | free slot | SDA :14007 (ASM3 slot) + another tap |
- The sensor is the TCP **client**; the harness never dials out [verified: Scout sessions via tap].
- Test PC is on DHCP (192.168.1.67 on 2026-11-26) → host and port must be config values.
- One SDA per sensor: the SDA keeps a single Registration [verified: SapientDataAgentClientProtocol.cs, ProcessRegistration].
- Go through the tap: it adds logging and checks, and Python already has a Windows firewall allow rule on the test PC;
  the SDA ports (1400x) have no firewall rule, so connecting to them from the LAN is likely blocked [verified: firewall rules 2026-11-26].

## Session the harness expects
1. Connect, then send **Registration first on every connection**. The SDA drops StatusReports and DetectionReports
   from a node_id that has not registered with it ("ASM ID not registered") [verified: SapientDataAgentClientProtocol.cs].
2. Wait for RegistrationAck: `acceptance: true`, reason "Accepted", `node_id` = DMM `71d47fbf-984d-4d4b-aca3-1d496ba1d96e`,
   `destination_id` = your node_id. Arrives in 0.3–0.5 s [verified: tap captures]. No ack within ~15 s → reconnect and
   register again (e.g. DMM simulator not started yet) [inferred].
3. StatusReport at the `status_interval` you declared. The SDA treats a client as gone after 60 s without data
   (`DisconnectIntervalSecs`) [verified: SapientDataAgent/Program.cs].
4. DetectionReports as detections arrive. No start Task is needed: the Scout sent detections without receiving any Task [verified: tap capture].
5. Handle what comes back: RegistrationAck, Task (answer with TaskAck), Error (see below), possibly AlertAck.
6. Before a planned disconnect send a StatusReport with `system: SYSTEM_GOODBYE`. Reconnect with backoff and re-register.

## Validation feedback loop
The DMM simulator runs with validation ON: an invalid message is **not processed** and an `error` message is sent back
to the sender (`destination_id` = your node_id, `error_message[]` = reasons, `packet` = your message bytes)
[verified: SapientServices/ClientBase.cs; routed to the sensor by DmmDataAgentTaskingProtocol.cs]. Log every Error you receive.

## Rules the harness enforces
- Envelope: `timestamp` = UTC send time; `node_id` = UUID that never changes for this sensor; `destination_id` optional.
- `report_id`, `object_id`, `task_id`, `region_id`: ULIDs (26 Crockford base32 chars, first char 0–7).
  The SDA parses every report_id as a ULID: one invalid report_id makes it drop that report and every later one of the same type
  [verified: Ulid.Parse in SapientDataAgentClientProtocol.cs]. Make report_ids increase with time (else "Non Consecutive" log).
- `icd_version` exactly `"BSI Flex 335 v2.0"` [verified: RegistrationValidator.cs].
- **Numeric zero quirk**: numbers checked with FluentValidation `NotEmpty` reject `0`: ENU `east_rate`/`north_rate` = 0.0 and
  Duration `value` = 0 fail [verified: ENUVelocityValidator.cs, DurationValidator.cs]. Omit the velocity for a stationary object;
  if exactly one component is 0.0 use a tiny non-zero value. Harness quirk, not an ICD rule [inferred].
- Registration must have: `node_definition` (node_type ≠ UNSPECIFIED), `icd_version`, `capabilities` (category + type),
  `status_definition.status_interval`, `mode_definition` (mode_name, mode_type, settle_time, task with **`concurrent_tasks`
  set explicitly** and `region_definition` with region_type + region_area units/datum), `config_data` (manufacturer, model).
  If present: commands need units + completion_time + type; detection_report definitions need category + type + units;
  class definitions need type, and each sub_class needs type + `level` [verified: Validation/*.cs].
- DetectionReport must have: report_id, object_id, exactly one of `location` / `range_bearing` (location: x, y, coordinate_system,
  datum; z optional). `detection_confidence` 0–1; object_info/track_info type + value non-empty; classification type non-empty,
  sub_class level set, confidences 0–1 [verified: DetectionReportValidator.cs and children].
- StatusReport must have: report_id, system, info, mode [verified: StatusReportValidator.cs].
- Clocks NTP-synced: the SDA computes latency from the envelope timestamp; the tap warns at > 5 s skew.

## Classification names
The harness carries a taxonomy (matched case-insensitively, NOT enforced for DetectionReports — only a non-empty type is checked)
[verified: SapientServices/Data/Validation/CommonValidator.cs]. Use its names until we read the standard's §7.2:
- `Air Vehicle` → `Manned Fixed Wing` | `Manned Rotary Wing` | `UAV Fixed Wing` | `UAV Rotary Wing` | `UAS Control Station` → `Commercial` | `Military`
- `Equipment` → `RF Transmitter` → `UAS Control Station`, `UAV Transmitter`, …; also `Unknown`, `Human`, `Animal`, `Land Vehicle`, `Sea Vessel`, `Fixed Object`.
- First sub-class has `level: 1`, the next `level: 2`.

## A Registration the harness accepts (the Scout's, verified 2026-11-26)
NODE_TYPE_PASSIVE_RF / sub_type "RemoteID"; capabilities [{Receiver, RemoteID}]; status_interval 60 s; one mode "default"
(MODE_TYPE_PERMANENT, settle_time 60 s, maximum_latency 2 s, SCAN_TYPE_FIXED, TRACKING_TYPE_TRACK_WITH_RE_ID,
detection_definition: location LAT_LNG_DEG_M + WGS84_G, velocity ENU m/s + WGS84_G; task: concurrent_tasks 1,
region_definition [REGION_TYPE_IGNORE] with area LAT_LNG_DEG_M + WGS84_G, command [{units "Registration",
completion_time 1 s, COMMAND_TYPE_REQUEST}]); config_data [{Dronetag, Scout}]. Copy the shape, not the values.

## Tasks the harness can send (DMM window → "Send Task", SensorID = your node_id, pick from the dropdown)
`task_id` ULID, `control` CONTROL_START/STOP/PAUSE, `command` oneof: `request` ("Registration", "Status", "Start", "Stop", "Reset" —
all sent with CONTROL_START), `detection_threshold` / `detection_report_rate` / `classification_threshold` (LOW/MEDIUM/HIGH),
`mode_change`, `look_at`, `move_to`, `patrol`, `follow`; templates also carry a sample region [verified: SapientDmmSimulator/Default.Task.*.json].
Reply to every Task with TaskAck: same `task_id`, `task_status` ACCEPTED | REJECTED | COMPLETED | FAILED, `reason[]` (say why on REJECTED).
Declare in the Registration only the commands you implement.

## Do better than the Scout (see sensors/scout.md)
- Keep reporting a detected object while it is detected, **even when it is not moving** (the Scout stops, so a C2 loses the track).
- Make the declared `status_interval` equal the real cadence (Scout: declares 60 s, sends every 10 s).
- `info`: INFO_NEW when the status content changed, INFO_UNCHANGED otherwise (Scout always sends INFO_NEW).
- Put the node's real position in StatusReport `node_location` (x = longitude, y = latitude).

## Acceptance test
1. On the test PC: `Start DMM.bat` first, then the sensor's DataAgent bat, then its tap; then start the sensor.
2. Pass: RegistrationAck received; no Error messages back; DMM window shows the messages with no "Error"; tap shows no warnings;
   SDA log has no "not registered" or "Internal" lines.
3. Evidence: tap JSONL (`Tools/SapientTap/logs/` in the harness repo); `Log/Log-SDA <port>/log.txt`, `Log/Log-HDA/log.txt`,
   `Log/Log-DMM/dmmlog.txt` in the deployed harness folder. Record the result in `decisions.md`.
