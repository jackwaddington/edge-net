# Handover — Edge-NET AV/AVR nodes

## What this is

Adding AV receivers into Edge-NET as a node class, with Home Assistant sitting
on the same MQTT fabric as a rich client. Two physical units, one shared capability.

## Core design (from mqtt-contract.md D7)

- **Capability over device.** Bus talks to `avr`, not to the box. Topics:
  - `edge-net/avr/power`  `{"state":"on"}` — subscribe (command)
  - `edge-net/avr/input`  `{"source":"bd-dvd"}`
  - `edge-net/avr/volume` `{"level":42}` or `up`/`down`
  - `edge-net/avr/mute`   `on`/`off`/`toggle`
  - `edge-net/avr/sound`  sound field (e.g. `pureDirect` for DJing)
  - `edge-net/avr/state`  publish, retained — actual sensed state
- **Two units at once → instance id:** `edge-net/avr/sony/*`, `edge-net/avr/hk/*`. Single unit omits it.
- **`/state` is earned.** Only publish when device truly senses.

## Two units, two transports

| Repo | Unit | Transport | `/state`? | Status |
| --- | --- | --- | --- | --- |
| edge-net-sony-dn1080 | Sony STR-DN1080 | Audio Control API on port 10000 (no auth) | **Yes** | Adapter written + deployed as LXC 210 on pve1 (192.168.0.60) |
| edge-net-hk-avr365 | Harman Kardon AVR365 | RS-232 via Pico W + MAX3232 | only if RxD wired | Scaffold only — decision pending |

## Sony — resolved (2026-06-22)

- **API confirmed:** `http://192.168.0.6:10000/sony/*`, no auth, open on LAN
- **Source URIs** (from live unit): `extInput:tv`, `extInput:bd-dvd`, `extInput:game`, `extInput:sat-catv`, `extInput:btAudio`, `extInput:sacd-cd`, `extInput:video?port=1/2`
- **Volume:** zone 1 range 0–55, zones 2+4 present (passthrough)
- **Adapter:** `edge-net-sony-dn1080/adapter/adapter.py` — polls every 5s, publishes `edge-net/avr/state` retained, handles all command topics
- **Deployed:** LXC CT 210 at 192.168.0.60, systemd service `edge-net-avr`, auto-start

## Home Assistant

- **VM 211** on pve1, HAOS 18.0, 2 cores / 4GB RAM / 32GB disk
- Provisioned via Terraform IaC (`infrastructure/terraform/environments/pve1/`)
- **After first boot:** open `http://homeassistant.local:8123`, run onboarding:
  1. Add MQTT integration → broker `192.168.0.145:1883`
  2. Add Sony STR-DN1080 natively (HA discovers it on the LAN)
  3. Optionally point `recorder:` at Postgres on db-01 for long-term history
- **HA + Edge-NET coexist:** HA publishes/subscribes same MQTT topics — nodes react to `edge-net/avr/state` without knowing HA exists

## DDJ-FLX4 DJ preset (planned)

Pioneer DDJ-FLX4 → RCA → Sony analog input (confirm which port — video1 or video2).
When DJing, Sony DSP adds 50–200ms latency — bypass with Pure Direct mode:

```text
edge-net/avr/input  {"source": "video1"}
edge-net/avr/sound  {"mode": "pureDirect"}
```

Wire as a one-button automation in HA once HA is set up.

## HK AVR365 — RS-232 protocol

Source: [HK AVR RS232 Protocol v1.0](https://github.com/jherland/hifictl/blob/master/HK%20AVR%20RS232%20Protocol.txt) (applies to all HK AVR/DPR units with RS-232 port)

### Serial settings

| Parameter | Value |
| --- | --- |
| Baud rate | 38,400 bps |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |
| Min gap between commands | 50 ms |

### Command frame (PC → AVR)

```text
"PCSEND" + 0x02 + 0x04 + [byte1] + [byte2] + [byte3] + [byte4] + checksum_hi + checksum_lo
```

Checksum: XOR of even-indexed bytes = high byte; XOR of odd-indexed bytes = low byte.
Volume/tuning commands have no repeat — send the command N times for N steps.

### Status frame (AVR → PC)

```text
"MPSEND" + 0x03 + 0x30 + [48 bytes]
```

48-byte payload contains two lines of 14-char front-panel display text + icon flags.
This means we can mirror the HK's display in the GUI exactly.

### Command table

| Command | Byte 1 | Byte 2 | Byte 3 | Byte 4 |
| --- | --- | --- | --- | --- |
| Power On | 80 | 70 | C0 | 3F |
| Power Off | 80 | 70 | 9F | 60 |
| Mute | 80 | 70 | C1 | 3E |
| Vol Up | 80 | 70 | C7 | 38 |
| Vol Down | 80 | 70 | C8 | 37 |
| AVR (input) | 82 | 72 | 35 | CA |
| DVD | 80 | 70 | D0 | 2F |
| CD | 80 | 70 | C4 | 3B |
| Tape | 80 | 70 | CC | 33 |
| VID1 | 80 | 70 | CA | 35 |
| VID2 | 80 | 70 | CB | 34 |
| VID3 | 80 | 70 | CE | 31 |
| VID4 | 80 | 70 | D1 | 2E |
| VID5 | 80 | 70 | F0 | 0F |
| AM/FM | 80 | 70 | 81 | 7E |
| 6CH/8CH | 82 | 72 | DB | 24 |
| Dolby | 82 | 72 | 50 | AF |
| DTS | 82 | 72 | A0 | 5F |
| DTS Neo:6 | 82 | 72 | A1 | 5E |
| Logic 7 | 82 | 72 | A2 | 5D |
| Stereo | 82 | 72 | 9B | 64 |
| Direct (bypass DSP) | 80 | 70 | 9B | 64 |
| Surround | 82 | 72 | 58 | A7 |
| Night mode | 82 | 72 | 96 | 69 |
| Sleep | 80 | 70 | DB | 24 |
| Dimmer | 80 | 70 | DC | 23 |
| OSD | 82 | 72 | 5C | A3 |
| OSD Left | 82 | 72 | C1 | 3E |
| OSD Right | 82 | 72 | C2 | 3D |
| Multiroom | 82 | 72 | DF | 20 |
| Multiroom Up | 82 | 72 | 5E | A1 |
| Multiroom Down | 82 | 72 | 5F | A0 |
| Test Tone | 82 | 72 | 8C | 73 |
| Preset Up | 82 | 72 | D0 | 2F |
| Preset Down | 82 | 72 | D1 | 2E |
| Tune Up | 80 | 70 | 84 | 7B |
| Tune Down | 80 | 70 | 85 | 7A |
| Memory | 80 | 70 | 86 | 79 |
| Clear | 82 | 72 | D9 | 26 |

## Pi 400 controller node (planned)

A **Raspberry Pi 400** (Pi 4 silicon, 4 GB RAM, built-in keyboard) acts as a TV-connected control surface and MQTT client. It does NOT replace the Pico W — the Pico W + MAX3232 remains the right adapter for the HK AVR365 (always-on, wired directly to the unit). The Pi 400 is a display/control client that sits on the same MQTT fabric alongside Home Assistant.

### Roles the Pi 400 plays

| Role | How |
| --- | --- |
| LG TV controller | `aiowebostv` over LAN; Wake-on-LAN for power-on |
| Display surface | HDMI → TV, custom full-screen GUI |
| Control surface | Built-in keyboard; IR receiver on GPIO for remote; web UI for couch |
| MQTT client | Subscribes to `edge-net/avr/state`, `edge-net/tv/lg/state` — does not own the adapters |

### Keyboard macros (examples)

- **Space × 3** → wake LG TV (WoL), switch to Pi HDMI input, show controller GUI
- **Single key** → switch AVR input, adjust volume, select sound mode

### LG TV control

Python library: `aiowebostv`. Power-on requires WoL (MAC TBD). All other controls (input, volume, mute, app launch) are over LAN via webOS API.

### MQTT topics added by this node

| Topic | Direction | Payload |
| --- | --- | --- |
| `edge-net/avr/hk/*` | pub/sub | same schema as sony topics |
| `edge-net/tv/lg/power` | pub/sub | `{"state":"on/off"}` |
| `edge-net/tv/lg/input` | pub | `{"source":"hdmi1"}` etc. |
| `edge-net/tv/lg/state` | pub retained | sensed state |

## Open questions

- Does Sony External Control API respond when unit is in standby?
- Which Sony analog input (video1 or video2) does the DDJ-FLX4 RCA go into?
- LG TV: confirm model and MAC address for WoL
- Which HDMI input on LG does the Pi 400 connect to?
- Make hub home-LAN IP static: ASUS router → DHCP reservation, MAC `d8:3a:dd:8e:c1:42`
- Make Sony IP static: ASUS router → DHCP reservation at 192.168.0.6
