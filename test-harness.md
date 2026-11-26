# Dstl SAPIENT Test Harness (v5.2.4, BSI Flex 335 v2 only)

Repo: github.com/dstl/BSI-Flex-335-v2-Test-Harness. Status: installed and working (2026-11-26).

## Constraints [verified: repo README]
- Windows only; .NET 6 SDK; **PostgreSQL 12 exactly** (later versions don't work).
- **English (UK/US) locale required** — on other locales numbers like 0.3 become 0,3 and break.
- Legacy code with SQL-injection risk → closed LAN, trusted users only. .NET 6 / PostgreSQL 12 are out of support [inferred: general knowledge] → isolated PC.
- Config split across several files; reinstall from clean copy if network config gets confusing. No support from Dstl.

## TCP topology [verified: BuildExecutableFolder configs, BatchFiles/*.bat, SapientDataAgent/Program.cs]
Data Agents are servers; simulators/sensors are clients.
| port | listener | client | notes |
|---|---|---|---|
| 14001 | SDA (sensor-side Data Agent), default | a sensor | `SDAClientPort` |
| 14005/14006/14007 | SDA for ASM 1/2/3 | ASM simulators | port passed as argument by start scripts, overrides config |
| 14000 | HDA (fusion-side Data Agent) | each SDA | `DACommunicationPort` |
| 12002 | HDA | DMM simulator (fusion node) | `HDATaskingPort` |
| 12003 | Data Agent (only with `G` argument) | GUI | `GuiPort` |
| 5432 | PostgreSQL | Data Agents | keep localhost-only |
- All addresses default to 127.0.0.1 → change bind addresses to the test PC's LAN IP; open firewall only on the closed LAN.
- Data Agents ship with `ValidationEnabled=False`, simulators with `True` → enable on the SDA to validate a real sensor [inferred; confirm in user manual].
- Harness pattern: one SDA per sensor.

## How we use it
1. Validate the Scout's native SAPIENT output.
2. Stand-in for c2lynx: sensors (our nodes) connect to an SDA; SDA → HDA → DMM simulator plays c2lynx.
3. Stand-in sensor: point an ASM simulator (`SapientDataAgentAddress/Port`) at our SAPIENT server.
4. Later: validate c2lynx itself.
Milestone acceptance gate, not CI. Log results in `decisions.md`.
