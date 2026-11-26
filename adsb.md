# ADS-B (Raspberry Pi)

## Existing service — do not modify
readsb + own MQTT publisher on the Pi, source_id `pi5-adsb-01`. The new SAPIENT node must not modify,
restart or reconfigure it; it consumes its output read-only.

## Observed data [verified: received-2026-11-10.jsonl, health-2026-11-10.jsonl]
- Aircraft topic `c2lynx/adsb/v1/pi5-adsb-01/aircraft`; envelope fields `kind`, `source_id`, `sent_at`.
- Payload fields include: address (ICAO), callsign, latitude, longitude, altitude_baro_m, altitude_geom_m,
  ground_speed_kmh, track_deg, vertical_rate_ms, squawk, on_ground, emergency, rssi_dbfs, nic, nac_p, sil, source_type.
- Only 95 of 198 aircraft messages had a position; many `source_type: mode_s` without lat/lon (normal).
- 8 messages also appeared on `cuas/espacio-aereo` from 16:31 — second namespace, unexplained.
- Health: `readsb` reported `activating` in 373 of 545 heartbeats, stuck for long periods (08:08–09:22, 09:22–10:12, 10:13–11:15)
  → check `journalctl -u readsb`. `clock_synced: true`.

## Plan: ADS-B as a SAPIENT sensor node (draft, not decided)
- Input: the existing MQTT aircraft feed (read-only).
- Output: own SAPIENT node (own stable node_id) connecting as client to the fusion node (harness SDA now, c2lynx later).
- Mapping draft [inferred]: ICAO → `id`; stable ULID per ICAO → `object_id`; position → Location with datum
  (geometric altitude → WGS84_E; barometric → object_info); NACp/NIC → location errors; callsign, squawk, RSSI,
  on_ground, emergency → object_info declared in Registration; classification from the §7.2 taxonomy (read the standard).
- Mode S messages without position: decide whether to report (range-less) or skip — open.
- Validate against the Dstl harness before c2lynx.
