# Decision and status log (newest first)

Format: `YYYY-MM-DD – topic – decision/status – source`

- 2026-11-26 – Knowledge sharing – shared `cuas-knowledge` repo (no shared code); each code repo imports its README; Claude.ai Project syncs it from GitHub.
- 2026-11-26 – Scout SAPIENT drops – localised to the Scout's SAPIENT output (its own per-protocol counter: SAPIENT rising, MQTT constant). Hypotheses and next tests in sensors/scout.md.
- 2026-11-26 – Scout MQTT + HTTP – both working simultaneously, continuous detections.
- 2026-11-26 – Test Harness – installed and working; topology and constraints in test-harness.md.
- 2026-11-26 – SAPIENT framing – confirmed in Dstl harness code: 4-byte LE length + protobuf.
- 2026-11-26 – Own SAPIENT fusion-node server – being built (PC agent); contract in sapient.md.
- 2026-11-26 – Upstream connection model – option A: one SAPIENT connection per sensor node, same c2lynx port.
- 2026-11-26 – Pi as SAPIENT relay – Scout and ADS-B each a SAPIENT node towards c2lynx; ADS-B consumed read-only from MQTT.
- 2026-11-26 – SAPIENT adoption – evaluate as output to c2lynx; keep MQTT until c2lynx supports SAPIENT.
- earlier – Scout gateway repo – Scout-only scope; dual ingestion MQTT + HTTP into one adapter; raw logging; dedup (UASID, Location.Timestamp) provisional.
