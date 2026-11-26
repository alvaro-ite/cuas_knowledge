# ADS-B (Raspberry Pi)

## Existing service
readsb + own MQTT publisher (`main.py`, systemd `c2lynx-adsb`) on the Pi, source_id `pi5-adsb-01`, serial `PI5-ADSB-01`.
Current topics `cuas/devices/PI5-ADSB-01/data` and `/health` [verified: pi-config/config.json 2026-11-26];
the `c2lynx/adsb/v1/...` topics below are from the 2026-11-10 captures and no longer used.

## SAPIENT node (implemented 2026-11-26, off by default, not yet tested against the harness)
- Runs inside `main.py` (repo folder `sapient/`): every aircraft record and health message published on MQTT is
  also sent over SAPIENT, so content and timing match MQTT [verified: Raspberry_FlightRadar sapient/client.py].
- node_id **`c6992234-fd97-4959-95c3-fd06f68be85a`** (fixed). Target: harness tap TCP :14101. Switch: `sapient.enabled`.
- Registration: NODE_TYPE_PASSIVE_RF / "ADS-B", status_interval 30 s (= real health cadence), commands: request
  "Registration" and "Status" only (others → TaskAck REJECTED).
- DetectionReport: `id` = ICAO, one `object_id` ULID per aircraft while tracked, location LAT_LNG_DEG_M + WGS84_E
  (z = altitude_geom_m), ENU velocity omitted when speed is 0, confidence 0.9, class `Air Vehicle`, rest in object_info.
  Records without position are skipped. StatusReport per health message, INFO_UNCHANGED when unchanged, node_location set.
- Not queued while disconnected; after each RegistrationAck: last StatusReport + current aircraft once. GOODBYE on stop.

## Observed data [verified: received-2026-11-10.jsonl, health-2026-11-10.jsonl]
- Aircraft topic `c2lynx/adsb/v1/pi5-adsb-01/aircraft`; envelope fields `kind`, `source_id`, `sent_at`.
- Payload fields include: address (ICAO), callsign, latitude, longitude, altitude_baro_m, altitude_geom_m,
  ground_speed_kmh, track_deg, vertical_rate_ms, squawk, on_ground, emergency, rssi_dbfs, nic, nac_p, sil, source_type.
- Only 95 of 198 aircraft messages had a position; many `source_type: mode_s` without lat/lon (normal).
- 8 messages also appeared on `cuas/espacio-aereo` from 16:31 — second namespace, unexplained.
- Health: `readsb` reported `activating` in 373 of 545 heartbeats, stuck for long periods (08:08–09:22, 09:22–10:12, 10:13–11:15)
  → check `journalctl -u readsb`. `clock_synced: true`.

## Original plan (draft, superseded by the section above where they differ)
- Input: originally the MQTT feed; implemented inside main.py instead.
- Output: own SAPIENT node (own stable node_id) connecting as client to the fusion node (harness now, c2lynx later).
- Harness target ready on the test PC: TCP :14101 (tap) → SDA :14006 (ASM2 slot, no database) [verified: set up 2026-11-26].
  How to connect, session, validation rules and acceptance test: `../sapient-sensor-node.md`.
- Node definition: NODE_TYPE_PASSIVE_RF, node_sub_type "ADS-B" (the Scout uses PASSIVE_RF / "RemoteID") [inferred choice].
- Mapping draft [inferred unless tagged]:
  - address (ICAO) → `id`; stable ULID per ICAO → `object_id`.
  - latitude/longitude → Location y/x (x = longitude), LAT_LNG_DEG_M.
  - altitude_geom_m → z with datum WGS84_E (ADS-B geometric altitude is ellipsoidal); altitude_baro_m → object_info.
  - ground_speed_kmh / 3.6 and track_deg → ENU east = v·sin(track), north = v·cos(track); vertical_rate_ms → up_rate.
    The harness rejects an east/north rate of exactly 0.0 → omit the velocity when stationary [verified: ENUVelocityValidator.cs].
  - NACp/NIC → location errors; callsign, squawk, rssi_dbfs, on_ground, emergency, nic, nac_p, sil, source_type → object_info,
    each declared in the Registration (detection_report definitions, category OBJECT).
  - classification `Air Vehicle`, sub-class `Manned Fixed Wing` / `Manned Rotary Wing` when the emitter category allows it
    (harness taxonomy names, see `../sapient-sensor-node.md`); detection_confidence < 1.0 (spoofable).
- Mode S messages without position cannot become a valid DetectionReport: location or range_bearing is mandatory
  [verified: DetectionReportValidator.cs] → skip them (or a later decision on a range-less report).
- Report every new position of every aircraft, including aircraft on the ground or hovering (lesson from the Scout).
- When an aircraft stops being received, optionally send a last report with `state` "lost" [inferred: proto comment].
- Validate against the Dstl harness before c2lynx.
