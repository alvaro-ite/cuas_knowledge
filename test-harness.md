# Dstl SAPIENT Test Harness (v5.2.4, BSI Flex 335 v2 only)

Repo: github.com/dstl/BSI-Flex-335-v2-Test-Harness. Status: installed and working on the Windows test PC (2026-11-26),
real Scout connected through it. Building a sensor that connects to it: `sapient-sensor-node.md`.

## Constraints [verified: repo README unless tagged]
- Windows only; .NET 6 SDK (6.0.428 installed); PostgreSQL 12 exactly if a database is used.
- **We run without PostgreSQL**: empty `DatabaseName` in a Data Agent config → "Running without Database"; database features
  (offsets, history) are off [verified: SapientDataAgent/Program.cs]. Done for the ASM1, ASM2 and DMM Data Agents.
- English locale required. On the es-ES test PC this is fixed by a local code change that pins en-GB in all three executables
  (`SapientServices/CultureSetup.cs`, not upstream) [verified: build + latency logs with "." decimals, 2026-11-26].
- Windows Smart App Control blocks the locally built DLLs (error 0x800711C7, unit tests fail, apps hang) → it is off on the test PC [verified: 2026-11-26].
- Legacy code with SQL-injection risk → closed LAN, trusted users only. .NET 6 / PostgreSQL 12 are out of support [inferred: general knowledge] → isolated PC.
- Config split across several files; reinstall from clean copy if network config gets confusing. No support from Dstl.

## TCP topology [verified: BuildExecutableFolder configs, BatchFiles/*.bat, SapientDataAgent/Program.cs]
Data Agents are servers; simulators/sensors are clients.
| port | listener | client | notes |
|---|---|---|---|
| 14100 | SapientTap → 14005 | Scout | our proxy, in use |
| 14101 | SapientTap → 14006 | ADS-B node | our proxy, ready |
| 14001 | SDA (sensor-side Data Agent), default | a sensor | `SDAClientPort` |
| 14005/14006/14007 | SDA for ASM 1/2/3 | ASM simulators or our sensors | port passed as argument by start scripts, overrides config |
| 14000 | HDA (fusion-side Data Agent) | each SDA | `DACommunicationPort` |
| 12002 | HDA | DMM simulator (fusion node) | `HDATaskingPort` |
| 12003 | Data Agent (only with `G` argument) | GUI | `GuiPort` |
| 5432 | PostgreSQL | Data Agents | not used (no database) |
- Data Agents listen on 0.0.0.0; only Python (the tap) has a Windows firewall allow rule → LAN sensors go through a tap [verified: 2026-11-26].
- One SDA per sensor: an SDA keeps a single Registration [verified: SapientDataAgentClientProtocol.cs].
- Start order: `Start DMM.bat` (HDA + DMM simulator) first, then the sensor's SDA (`Start Scout DataAgent.bat` → 14005,
  `Start ADSB DataAgent.bat` → 14006; local batch files), then the tap, then the sensor. `Start ASM n.bat` also starts a simulated sensor — don't use it for a real one.

## Behaviour that matters to a sensor
- DMM simulator: validation ON; auto-replies RegistrationAck (acceptance true, "Accepted", node_id `71d47fbf-984d-4d4b-aca3-1d496ba1d96e`);
  an invalid message is not processed and an `error` message (reasons + original packet) is sent back to the sender
  [verified: SapientServices/ClientBase.cs, Default.RegistrationAck.json, tap captures].
- Data Agents: validation OFF; they log "Message is invalid but validation ignored" + reasons and forward
  (reasons visible only with a local logging fix in ClientBase.cs) [verified].
- SDA drops reports from a node_id that has not registered with it; drops all reports after a non-ULID report_id;
  treats a client as gone after 60 s without data (`DisconnectIntervalSecs`); computes latency from the envelope timestamp [verified: SDA code].
- Validators reject numeric 0 where they use `NotEmpty` (ENU east/north rate, Duration value) [verified: Validation/*.cs].
- Full sensor-facing rule list: `sapient-sensor-node.md`.

## SapientTap (local tool, harness repo `Tools/SapientTap/`, untracked)
Transparent TCP proxy in front of an SDA (Python, compiles the harness protos). Forwards bytes unchanged, decodes both directions,
prints JSON, writes JSONL per session (`logs/sapient_tap_<start time>.jsonl`), flags missing mandatory fields, bad UUIDs/ULIDs,
wrong icd_version, clock skew > 5 s and framing errors. `--dial` mode exists for sensors that are TCP servers.

## How we use it
1. Validate the Scout's native SAPIENT output (done 2026-11-26, see `sensors/scout.md`).
2. Stand-in for c2lynx: sensors (our nodes) connect to an SDA; SDA → HDA → DMM simulator plays c2lynx.
3. Stand-in sensor: point an ASM simulator (`SapientDataAgentAddress/Port`) at our SAPIENT server.
4. Later: validate c2lynx itself.
Milestone acceptance gate, not CI. Log results in `decisions.md`.
