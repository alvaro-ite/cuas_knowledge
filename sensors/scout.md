# Dronetag Scout

## Outputs
- MQTT: working, continuous detections. Topic `cuas/devices/DronetagScout/data` [verified: Antena2.json];
  on 2026-11-26 the topic was `cuas/devices/D189C831D0375A719A/data` (serial number) → topic changed or configurable [verified: MQTT log pasted by user].
- HTTP: working, continuous detections. Route/format: record here [TBD by PC agent].
- MQTT and HTTP run simultaneously [verified: user, 2026-11-26].
- SAPIENT: native output; Scout connects as TCP client to the fusion node (harness SDA or our server).
  node_id `d05e1c85-95ce-55d5-9176-4813e8a43653`, constant across reconnects; name "Dronetag Scout D189C831D0375A719A",
  short_name "SCOUT-719A" [verified: Registration via SapientTap, 2026-11-26].

## Observed SAPIENT behaviour [verified: SapientTap logs sapient_tap_20261126-110519/-123604.jsonl, harness PC]
- Registration: NODE_TYPE_PASSIVE_RF / "RemoteID", icd "BSI Flex 335 v2.0", status_interval 60 s, mode "default",
  TRACK_WITH_RE_ID, only command COMMAND_TYPE_REQUEST "Registration", region REGION_TYPE_IGNORE, config_data Dronetag/Scout.
  Accepted by the harness, no validation errors.
- StatusReports every 10 s (not the declared 60 s), always `info: INFO_NEW`, `system: SYSTEM_OK`, fixed `node_location`
  (x = lon, y = lat, z 679.5 m, WGS84_G).
- Saving settings in the Scout web UI → StatusReport `SYSTEM_GOODBYE`, disconnect, reconnect and re-register within ~50 ms (6 sessions seen).
- Does not wait for a Task: detections flowed with only RegistrationAck sent to it.
- Clock ~1.7–1.9 s ahead of the harness PC (RegistrationAck timestamp earlier than the Registration it answers).
- DetectionReports come in pairs: aircraft (`Air vehicle` / `UAV rotary wing`, with enu_velocity, `id` = UAS ID) and
  control station (`EM transmitter` / `UAS control station`, no z, `associated_detection` SIBLING to the aircraft).
  object_ids are deterministic per object (syntactically valid ULIDs; the pair shares a suffix). Message timestamp = RID
  timestamp (0.1 s steps). object_info: Technology, Operator ID, UAS ID, Manufacturer, Model.
- Detected drone ~32 km from the Scout's reported `node_location` while MQTT RSSI was −38…−54 dBm (noise floor −96)
  → the Scout's configured position is probably wrong [inferred].
- MQTT `registration` field = drone maker/model lookup (e.g. DJI / M350 RTK), not SAPIENT [verified: MQTT log 2026-11-26].

## Observed MQTT payload facts [verified: Antena2.json]
- Same drone reported by several Scout modules (different `recv_id` / `module_type`, per-module `counter`) → dedup needed.
  Provisional key: (UASID, Location.Timestamp).
- `Location.Timestamp` has no timezone, with `TimestampEstimated: true`; treated as UTC [inferred, not confirmed].
- Altitudes inconsistent in sample (AltitudeBaro 105, AltitudeGeo 5, Height 5, HeightType 0) → keep all, don't correct.
- Fields with undocumented meaning: `msg_type` (15 seen), `module_type`, `tech` (`WB` seen). (`registration`: see below.)
- Test drone: UAS ID prefix `1581...` (likely DJI) received via `tech: WB` (likely Wi-Fi Beacon) [inferred].
  → iPhone RID apps (Bluetooth only on iOS) won't see it; use Android or a Bluetooth-emitting simulator for cross-checks.

## Issue: SAPIENT output drops detections (cause narrowed, confirmation pending)
- Symptom: Registration/StatusReports arrive; few DetectionReports. The Scout's own per-protocol drop counter:
  SAPIENT keeps rising, MQTT constant → loss is inside the Scout's SAPIENT output path [verified: user, 2026-11-26].
- Tested against the Dstl harness with SapientTap in front of it (2026-11-26):
  - Not the harness: all 44 DetectionReports the Scout sent reached the DMM simulator, 0 validation errors [verified: tap + harness logs].
  - Not the session/ack (old hypothesis 1, ruled out): every session got the same RegistrationAck (acceptance true,
    destination_id = Scout node_id); sessions with and without detections got identical acks; the Scout never re-registered within a session [verified: tap logs].
  - Not waiting for a start Task (part of old hypothesis 3, ruled out) [verified: detections arrived with no Task sent].
  - **Leading cause: the SAPIENT output only sends a detection while the target moves.** The 44 detections came in 3 bursts
    of 4–5 s during take-off, climb and descent; every sent aircraft detection had speed 0.25–5.5 m/s and a new position.
    At 11:42 UTC MQTT showed the same drone continuously (speed 0.0, position identical to 7 decimals) and the Scout sent
    0 SAPIENT detections in the same seconds [verified: tap log -123604 vs MQTT log]. Positioned-but-static RID messages are
    therefore not sent; the rising drop counter is probably these [inferred].
- Confirming test: move the drone (real GPS position change) and watch the tap: detections should start and stop with movement.
- Impact: a hovering or landed-but-powered drone disappears from a SAPIENT C2 although the Scout still receives it.
- If confirmed: ticket to Dronetag with per-protocol counters, firmware version, settings and the tap captures.

## Questions for Dronetag
Is there a setting to keep reporting stationary targets (minimum report rate / keep-alive)? Meaning of the drop counter;
declared status_interval 60 s vs actual 10 s; `info` always INFO_NEW; firmware known issues; ICDs for SAPIENT/MQTT/HTTP.
