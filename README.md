# cuas-knowledge — shared knowledge for the C-UAS sensor work

Shared, code-free knowledge base. Read by:
- the Claude Code agent on the PC (Scout gateway / SAPIENT server work),
- the Claude Code agent on the Raspberry Pi (ADS-B SAPIENT node work),
- the Claude.ai Project (synced from GitHub).

Each code repo stays independent; only knowledge lives here.

## Files (read the one relevant to your task; don't load everything)
| file | content |
|---|---|
| `system-overview.md` | components, roles, network, topics, ports, who talks to whom |
| `sapient.md` | SAPIENT (BSI Flex 335 v2.0) facts we verified, server and sensor behaviour contract, open questions |
| `test-harness.md` | Dstl Test Harness v5.2.4: constraints, TCP topology, how we use it |
| `sensors/scout.md` | Dronetag Scout: outputs, observed data, SAPIENT drop issue |
| `sensors/adsb.md` | existing ADS-B service on the Pi, observed data, plan for its SAPIENT node |
| `decisions.md` | dated decision and status log (newest first) |

## Rules for every agent
1. **Tag every fact with its source**: `[verified: <file/capture/code path>]`, `[doc: <document>]` or `[inferred]`.
   Never promote `[inferred]` to verified without evidence.
2. **When you change an interface or learn something another agent needs** (ports, node_ids,
   message content, a confirmed behaviour, a bug cause): update the relevant file here and add a
   line to `decisions.md`, in the same session. Commit and push this repo.
3. **Before starting work**: `git pull` this repo.
4. Don't put code, credentials, IPs of production systems or secrets here. Describe interfaces, not implementations.
5. Keep files short. Remove facts that are no longer true (and note it in `decisions.md`).

## Current status (2026-11-26)
- Scout → MQTT and HTTP: working, continuous detections.
- Scout → SAPIENT: Registration/StatusReports arrive, but the Scout's own SAPIENT drop counter keeps rising. Under investigation (`sensors/scout.md`).
- Own SAPIENT fusion-node server: being built on the PC agent.
- ADS-B SAPIENT node on the Pi: starting (`sensors/adsb.md`).
- Dstl Test Harness: installed and working.
- c2lynx SAPIENT support: not yet.
