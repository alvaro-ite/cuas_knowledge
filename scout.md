# Dronetag Scout

## Outputs
- MQTT: working, continuous detections. Topic `cuas/devices/DronetagScout/data` [verified: Antena2.json].
- HTTP: working, continuous detections. Route/format: record here [TBD by PC agent].
- MQTT and HTTP run simultaneously [verified: user, 2026-11-26].
- SAPIENT: native output; Scout connects as TCP client to the fusion node (harness SDA or our server).
  Scout SAPIENT node_id: TBD (record from its Registration).

## Observed MQTT payload facts [verified: Antena2.json]
- Same drone reported by several Scout modules (different `recv_id` / `module_type`, per-module `counter`) → dedup needed.
  Provisional key: (UASID, Location.Timestamp).
- `Location.Timestamp` has no timezone, with `TimestampEstimated: true`; treated as UTC [inferred, not confirmed].
- Altitudes inconsistent in sample (AltitudeBaro 105, AltitudeGeo 5, Height 5, HeightType 0) → keep all, don't correct.
- Fields with undocumented meaning: `msg_type` (15 seen), `module_type`, `tech` (`WB` seen), `registration`.
- Test drone: UAS ID prefix `1581...` (likely DJI) received via `tech: WB` (likely Wi-Fi Beacon) [inferred].
  → iPhone RID apps (Bluetooth only on iOS) won't see it; use Android or a Bluetooth-emitting simulator for cross-checks.

## Issue: SAPIENT output drops detections (open)
- Symptom: Registration/StatusReports arrive; few DetectionReports. The Scout's own per-protocol drop counter:
  SAPIENT keeps rising, MQTT constant → loss is inside the Scout's SAPIENT output path [verified: user, 2026-11-26].
- Hypotheses:
  1. Session not accepted by the Scout (RegistrationAck content: acceptance, destination_id, node_id).
  2. RID messages without position can't become valid DetectionReports and are counted as dropped (may be expected).
  3. Rate limit / mode / queue in the SAPIENT output; or waiting for a start Task.
- Next tests: point the Scout at the Dstl harness (does the counter stop?); counter with vs without the drone emitting;
  one SAPIENT setting at a time; send a start Task from our server.
- If Scout-side: ticket to Dronetag with per-protocol counters, firmware version, settings, harness result.

## Questions for Dronetag
Meaning of the drop counter; SAPIENT rate/mode limits; whether it waits for a Task or specific Ack; firmware known issues; ICDs for SAPIENT/MQTT/HTTP.
