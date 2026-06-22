# Handover — Edge-NET AV/AVR nodes

## What this is
Adding AV receivers (and TVs) to Edge-NET as a node class. Two physical units, one
shared capability on the bus. Repos scaffolded Jun 17; no adapter code yet.

## Core design (from mqtt-contract.md D7)
- **Capability over device.** Bus talks to `avr`, not to the box. Topics:
  - `edge-net/avr/power`  `{"state":"on"}` — subscribe (command)
  - `edge-net/avr/input`  `{"source":"bd-dvd"}`
  - `edge-net/avr/volume` `{"level":42}` or `up`/`down`
  - `edge-net/avr/mute`   `on`/`off`/`toggle`
  - `edge-net/avr/sound`  sound field
  - `edge-net/avr/state`  publish, retained — **actual** sensed state
- **Two units at once → instance id:** `edge-net/avr/sony/*`, `edge-net/avr/hk/*`.
  Single unit omits it.
- **`/state` is earned, not given.** Build it only when the device truly senses
  (pushes real state on remote/front-panel change). Same rule as D4a relay / D6
  release-events: cheap-and-honest ships, expensive-or-theatre waits.

## Two units, two transports
| Repo | Unit | Transport | `/state`? | Status |
| --- | --- | --- | --- | --- |
| edge-net-sony-dn1080 | Sony STR-DN1080 | network-native, Audio Control API (TCP/UDP 33336, JSON-RPC) | **Yes** — API pushes notifications | scaffold: README + CONTROLS.md, capability topics adopted (commit 38f4465). External Control not yet enabled, adapter not written |
| edge-net-hk-avr365 | Harman Kardon AVR365 | RS-232 via Pico W + MAX3232 | **only if** RxD wired to read 48-byte status frames; else presence-only | scaffold: README + RS-232 controls catalog |

Sony = **adapter + catalog**, nothing runs on the box (it's on the LAN). Adapter
service lives on a Linux node/VM, bridges MQTT ↔ Audio Control API. HK = needs
bridge hardware to reach RS-232.

## Sony open questions (from CONTROLS.md)
- Exact source-name strings the API expects (capture from live unit).
- Does External Control survive standby, or only when powered? (test)
- Model Zone 2 as own capability vs nested? (Zone 2 = the whole-house lever:
  main = lounge, Zone 2 = another room, independently switchable.)
- Auth/pairing: does DN1080 need a handshake or is it open on LAN?

## Next steps (suggested order)
1. Enable Sony External Control, confirm port 33336 responds.
2. Capture real source-name strings + a state notification sample.
3. Write Sony adapter: MQTT `edge-net/avr/*` ↔ Audio Control API, publish retained
   `/state` from API notifications.
4. HK: decide if RxD read worth wiring (earns `/state`) or presence-only for now.

## Key files
- Contract: edge-net/docs/mqtt-contract.md §D7 (lines ~166-206)
- Sony: edge-net-sony-dn1080/README.md, CONTROLS.md
- HK: edge-net-hk-avr365/README.md, CONTROLS.md
- Principles in play: "capability over device", MQTT boundary, "build richer
  contract when the device truly senses".
