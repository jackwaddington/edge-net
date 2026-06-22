# Edge-NET — MQTT contract

The shared language between nodes. MQTT is the boundary: below it, rigid and
reliable device behaviour; above it, anything reacts to anything. Every node
repo conforms to this contract so nodes never need to know about each other.

This documents the *vocabulary*, not the *thinking*. There is no intent engine
yet — see [PLANNING.md](PLANNING.md). We are defining how nodes talk, not what
they should decide.

---

## Topic structure

```text
edge-net/<node>/<category>/<id>
```

- `<node>` — the node name without the `edge-net-` repo prefix: `keybow`,
  `gfx`, `plasma`, `automation`, `gamepad`, `kindle`, `inky`. The AV/media
  nodes (`hk-avr365`, `sony-dn1080`, `lg-um7400`, `samsung-frame`) are
  addressed by a **capability alias**, not their repo name — see D7.
- `<category>` — the kind of IO: `button`, `axis`, `led`, `relay`, `display`,
  `status`, …
- `<id>` — which one: `0`, `a`, `x`, `relay/1`. Omitted where there's only one
  (`edge-net/gfx/display`, `edge-net/plasma/led`).

**Direction follows the principle "inputs publish, outputs subscribe":**

- Inputs (buttons, axes, sensors) **publish** events — a node tells the network
  something happened.
- Outputs (LEDs, relays, displays) **subscribe** to commands — a node is told
  what to do.

A node never addresses another node directly; it publishes to its own topic and
whatever cares reacts.

## Payload format (D2)

**JSON is the baseline** — flat, minimal objects. Most nodes are full Linux or
can afford a small parser, so the consistency, extensibility, and
database-friendliness of JSON win. Examples:

```text
edge-net/keybow/button/0   {"event": "press"}
edge-net/keybow/led/0      {"rgb": [255, 0, 0]}
edge-net/automation/relay/1 {"state": "on"}
edge-net/gfx/display       { ...layout spec... }
```

**Exception — high-rate continuous streams** use a compact encoding (a bare
number), not JSON. This is the gamepad joystick axes and the Automation ADC
channels: they publish many times a second, and a per-sample JSON allocation
hurts the weakest node (Pico W, 264 KB RAM). Optimise the format for the common
case; special-case the one hot path rather than dumbing the whole network down.

```text
edge-net/gamepad/axis/x    512
edge-net/automation/adc/0  734
```

Rule: **discrete events → JSON; continuous telemetry → bare number.**

Timestamps, sequence numbers, and source metadata are *not* added by the dumb
device — the consumer stamps on receipt. Keep the wire payload to the physical
fact.

## Node liveness (D3)

Presence is tracked with MQTT's Last-Will-and-Testament (LWT), not polling.
Every node uses `edge-net/<node>/status`, **retained**:

- On connect, the node publishes its online state:
  `{"status": "online", "ip": "10.1.1.10", "fw": "1.2.0"}`
- It registers an LWT with the broker: `{"status": "offline"}` on the same
  topic, retained. If the node drops (crash, power cut, WiFi loss) the **broker**
  publishes this for it — automatically, no heartbeat anywhere.

Because the topic is retained, anything subscribing to `edge-net/+/status` gets
the current state of the whole network immediately — a health panel is free.

**The will is frozen at connect time**, so it carries nothing dynamic — just
`status: offline`. The richer fields (`ip`, `fw`) live only in the online
message, because they're known at connect and useful for debugging and for
spotting a node running stale firmware (see the update model in PLANNING.md).

**Liveness is not metrics.** Uptime, CPU, and memory are time-series and belong
to Prometheus (the GFX → Grafana path), not the status topic. Don't heartbeat
metrics onto the bus — that rebuilds a worse Prometheus. LWT = one retained
state; Prometheus = the stream.

## Retention and state restore (D4)

**Retain state, never retain events.**

- **State topics** (`led`, `relay`, `display`, `status`) are published
  **retained**. When a node reboots and resubscribes, the broker immediately
  replays the last command and the node restores itself — the Plasma strip comes
  back to its colour after a power blip, for free. Same mechanism as liveness.
- **Event topics** (`button`, `axis`) are **never retained**. A retained
  `press` would re-fire on every reconnect — a phantom event. These are
  fire-and-forget.

### One uniform behaviour, until a device can sense reality (D4a)

All output devices behave the same way: a single retained command topic where
**command = state**. Relays are *not* a special case, even though they switch
real-world power.

The "robust" alternative — a set/state split (`…/set` command + `…/state` the
device reports back) — exists so an actuator can report *what actually
happened*, not just what it was told. But that only carries information if the
device can **independently sense its own output**. Today neither an LED nor an
Automation Hat relay can: their `/state` would just echo the command — same
information, new topic. That is confirmation theatre, and worse than nothing
because it implies a guarantee that isn't there.

So the dividing line is **not device type and not "be strict everywhere"** — it
is **does this device have feedback?** Add a `/state` topic to a specific device
only when it gains real sensing (a current sensor on a relay, a reed switch
confirming a latch threw, an ADC on the load). Then `/state` is honest. Until
then, uniform retained command topics for everything. Strictness arrives with
sensors, not with categories.

## Delivery guarantees / QoS (D5)

Constraint: MicroPython `umqtt.simple` supports only QoS 0 and 1 — there is no
QoS 2 on the Pico nodes. Exactly-once is achieved at the consumer, not the
protocol.

| Topic class | QoS | Retained | Why |
| --- | --- | --- | --- |
| Continuous (`axis`, `adc`) | 0 | no | high-rate; a lost sample is superseded instantly |
| Events (`button`) | 1 | no | must not be lost; duplicates tolerated |
| State (`led`, `relay`, `display`) | 1 | yes | command must land, latest wins, restores on reboot |
| Status (LWT) | 1 | yes | presence must be reliable |

The trade-off is `button` at QoS 1: at-least-once can deliver a press twice, and
a double-logged chore is wrong. The fix is **not** QoS 2 (unsupported and heavy)
but **idempotency at the consumer** — dedupe on a message id or debounce by time
window. The one smart consumer above the bus owns exactly-once; the dumb nodes
never do.

## Button events (D6)

Every button publishes **both edges**, not just press:

```text
edge-net/keybow/button/0   {"event": "press"}
edge-net/keybow/button/0   {"event": "release"}
```

Release is what unlocks hold semantics — hold-to-confirm, long-press, chords —
none of which are detectable from press alone. It costs one extra line in
firmware, and the node genuinely senses the GPIO edge (not theatre). A consumer
that only wants taps ignores `release`; the richer contract subsumes the simpler
one, while press-only could never grow hold support without a reflash.

Note this is the opposite call to the set/state split (D4a), by the same test:
release is cheap *and* truly sensed → build it now; the set/state split was a
topic's cost *and* unsensable → defer it. Cheap-and-honest ships; expensive-or-
theatre waits.

(Existing nodes to align: Keybow and GFX currently emit `press` only.)

## Hub audio (D7)

The Pi 3A runs an audio service that plays sounds over its 3.5mm jack. Any node
can trigger audio by publishing to `edge/hub/audio/play` — no firmware changes
needed on the triggering node.

```text
edge/hub/audio/play   {"file": "chime.wav"}          # play a sound file
edge/hub/audio/play   {"tts": "motion detected"}     # speak text (espeak)
edge/hub/audio/play   {"volume": 70}                 # set master volume 0–100
edge/hub/audio/state  {"status": "playing", "source": "chime.wav"}  # pub, retained
edge/hub/audio/state  {"status": "idle"}             # pub, retained
```

Sound files live at `/opt/edge-net/sounds/` on the Pi 3A. `chime.wav` is
committed to `edge-net-automation` and deployed there. Messages queue — if two
arrive at once they play in order, never overlapping.

**Status: live** — `edge-net-automation/audio_service.py`, systemd unit
`edge-net-audio.service`, verified 2026-06-22.

## AV / media nodes (D8)

AV receivers and TVs are a new node shape: **command-driven outputs with a rich
verb set** (power, input, volume, surround, art). Two things make them distinct
from the LED/relay/display outputs above.

**Addressed by capability, not by box.** Per *capability over device*, the bus
talks to `avr` and `tv`, not to `hk-avr365` or `sony-dn1080`. Multiple physical
units can answer the same topic, and the edge fires one command regardless of
which transport (RS-232 via Pico, IP, WebSocket) carries it:

```text
edge-net/avr/power   {"state": "on"}
edge-net/avr/input   {"source": "dvd"}
edge-net/avr/volume  {"level": 42}        # absolute where supported, else up/down
edge-net/tv/art      {"state": "on", "image": "daily-render"}
```

When two units of one class run at once and must be steered independently, append
an instance id (`edge-net/avr/sony/power`, `edge-net/avr/hk/power`); a single unit
omits it. The repo name (`edge-net-sony-dn1080`) stays the catalogue of *what that
box can do*; the topic stays the *capability* anyone can drive. The transport is
the adapter's secret — exactly the MQTT boundary doing its job.

**These outputs genuinely sense their own state — so `/state` is honest here.**
D4a deferred the set/state split because an LED or relay can only echo its command
(confirmation theatre). The network-native AV nodes are the opposite: their APIs
**push notifications** on real state change — the user picks up the physical remote,
the front panel turns a knob, the TV's motion sensor fires — and the box reports it.
That is true sensing, so these nodes publish a retained `/state` reflecting *actual*
device state, separate from the command topic:

```text
edge-net/avr/power   {"state": "on"}      # subscribe — command
edge-net/avr/state   { ...actual power/input/volume... }   # publish — sensed, retained
```

Same test as release-events (D6) and the relay (D4a): build the richer contract
**when the device truly senses**, not by category. The HK bridge only earns `/state`
if its RxD line is wired to read the AVR's 48-byte status frames; until then it
publishes presence (`/status` LWT) but no `/state`.
