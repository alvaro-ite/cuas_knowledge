# Decision and status log (newest first)

Format: `YYYY-MM-DD – topic – decision/status – source`

- 2026-11-26 – Knowledge repo name – folder and GitHub repo are `cuas_knowledge` (underscore); an import of `../cuas-knowledge/README.md` loads nothing → use the underscore path in each repo's CLAUDE.md.
- 2026-11-26 – Sensor node guide – new `sapient-sensor-node.md`: ports, session, harness validation rules, acceptance test for our own sensor nodes. – harness code + captures.
- 2026-11-26 – ADS-B harness target – ASM2 slot prepared: SDA :14006 without database, tap :14101 (`Start ADSB DataAgent.bat`, `Start Tap ADS-B.bat`). – test PC.
- 2026-11-26 – Scout SAPIENT drops – harness and RegistrationAck ruled out; leading cause: Scout sends SAPIENT DetectionReports only while the target moves (static drone on MQTT, 0 SAPIENT detections). Confirm by moving the drone. – tap captures vs MQTT log.
- 2026-11-26 – Scout SAPIENT node_id – `d05e1c85-95ce-55d5-9176-4813e8a43653` (stable across reconnects). – Registration capture.
- 2026-11-26 – Test Harness setup – no PostgreSQL (empty DatabaseName), en-GB pinned in code, Smart App Control off, SapientTap proxy in front of each SDA. Replaced the [inferred] "enable validation on the SDA" note with verified behaviour (DMM validates and returns Error messages). – test-harness.md.
- 2026-11-26 – SAPIENT start/stop tasks – encoding answered from harness templates (`command.request` "Start"/"Stop", control START); removed from open questions. – Default.Task.Request.*.json.

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
