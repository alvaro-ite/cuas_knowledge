# System overview

## Components
| component | where | role | notes |
|---|---|---|---|
| Dronetag Scout | field, LAN | Remote ID receiver (BT4/5 LR, Wi-Fi Beacon/NAN) | outputs: MQTT, HTTP, SAPIENT (also Cloud v2, TAK, FlytBase, SafeSky — not used) |
| ADS-B service | Raspberry Pi (`pi5-adsb-01`) | readsb + own MQTT publisher | existing, independent service [verified: health capture 2026-11-10] |
| Scout gateway / SAPIENT server | PC agent's repo | receives Scout data; SAPIENT fusion-node server for testing | in development |
| ADS-B SAPIENT node | Pi agent's repo | turns ADS-B data into a SAPIENT sensor node | starting |
| c2lynx | command center | C2; today consumes JSON over MQTT | SAPIENT fusion-node support pending |
| Dstl Test Harness | Windows test PC, closed LAN | SAPIENT compliance + stand-in for c2lynx / sensors | see `test-harness.md` |

## Network
- Everything is on a **local network (LAN)**, not the internet. Clients connect to server IP:port.
- A server port accepts many clients; two server programs cannot listen on the same port of one host.
- `127.0.0.1` binds only to the same machine; LAN access needs the LAN IP or `0.0.0.0` plus firewall rules.
- Check ports before binding: `ss -ltnp`. Typical readsb/web ports to avoid on the Pi: 30002/30003/30005, 80/8080/8504 [inferred: general knowledge, not checked on this Pi].

## MQTT topics in use
| topic | publisher | source |
|---|---|---|
| `cuas/devices/DronetagScout/data` | Scout | [verified: capture Antena2.json] |
| `c2lynx/adsb/v1/pi5-adsb-01/aircraft` | ADS-B service | [verified: received-2026-11-10.jsonl] |
| `c2lynx/adsb/v1/pi5-adsb-01/health` | ADS-B service | [verified: health-2026-11-10.jsonl] |
| `cuas/espacio-aereo`, `cuas/espacio-aereo_health` | ADS-B service (same source_id) | [verified: same captures] — second namespace, origin unexplained |
- Proposed (not decided): RID output namespace `c2lynx/rid/v1/<node_id>/{uas,health}` mirroring the ADS-B one.

## SAPIENT connections (target)
- Each sensor is its own SAPIENT node with its own connection (decision: option A, see `sapient.md`).
- Ports: Scout-facing SAPIENT server port proposed **14001** (same as harness SDA default) so the Scout config only changes IP. c2lynx port: TBD.
