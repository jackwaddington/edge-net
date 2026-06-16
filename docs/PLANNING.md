# Edge-NET — Planning

Ideas, new nodes, and open questions. This is definition-first: describe what we want to build, challenge the assumptions, find what's possible — then extract code from here into the individual node repos.

---

## Philosophy

The environment should express what's going on. Not "look at your phone" — the room itself tells you. LEDs, a servo arrow, an e-ink display in the hallway. Physical buttons that log events without requiring a screen. The network is the substrate; the experience is ambient.

---

## Planned Nodes

### Kindle — Ambient E-ink Display

**Hardware**: Old Kindle, no backlight, d-pad + page-turn buttons (4th or 5th gen).

**Approach**: Jailbroken. Full SSH access, Python runtime, `eips` for direct e-ink writes. The Kindle connects to the Edge-NET WiFi AP; the hub's firewall decides what it can reach on the home network (a dashboard server that generates and serves a PNG).

**Display**: A service on the home network generates a PNG — task states, node health, whatever is useful — and serves it over HTTP. The Kindle polls it on a cron job and pushes it to the screen with `eips -g image.png`. Refresh every few minutes.

**Why jailbreak fully**: Full OS control. Disable screensaver, prevent sleep, control WiFi from scripts, use buttons as triggers if wanted later. The buttons exist; we're not building a button UI on the Kindle, but having them available is worth the jailbreak.

**Open questions**:
- What does the Kindle display? (task tracker, node health, home dashboard, all of the above?)
- Where does the dashboard server run? (Pi Zero W is already on the network and could serve it)

---

### Gnome — Servo Arrow Ambient Indicator

**Hardware**: Pi Pico W + SG90 micro servo (~£2). Decorative gnome ornament with an arrow.

**Approach**: Servo powered directly from Pico W 3.3V pin — no separate PSU. Single USB cable into the gnome. Pico W connects to Edge-NET WiFi, subscribes to an MQTT topic, sets servo angle on message.

**Display**: Arrow mounted on servo horn, pokes through the ornament. A small printed dial behind the gnome shows what the positions mean (e.g. labels around an arc).

**Why this is interesting**: Looks like a decorative object. Nobody knows there's a microcontroller in it. The arrow moves on its own in response to real events — genuinely uncanny in a good way.

**Open questions**:
- What does the arrow point at? What's the one thing it represents? (mood of the network, a person's status, a task state, something else?)
- Fixed position or rotates full 180°?

---

### Control Panel — Physical Config Surface

**Hardware**: Pi Pico W + Pimoroni Display Pack 2.0 (2.8" 320×240, 4 built-in buttons) + 2-3 rotary encoders.

**Approach**: Context-sensitive control surface. Buttons navigate a menu on the display; the active menu page determines what the knobs do. Turn a knob on the "temperature" page → publishes `edge/cmd/temperature/set`. Switch to "speed" page → same knob now controls speed. Fully decoupled — the panel publishes MQTT, whatever cares about that topic reacts.

**Why rotary encoders not potentiometers**: No fixed range — scroll through menus, increment values without hitting a physical limit. Push-click for confirm/select.

**Enclosure**: Open question — project box, wooden box, or 3D printed. The Display Pack is one clean unit (screen + 4 buttons integrated), which reduces the number of holes to cut.

**Open questions**:
- What is on the top-level menu? What are the parameters worth tuning physically?
- Fixed mount somewhere or handheld?
- Enclosure approach?

---

### Relay — Switching Physical Things

**Hardware**: Pi 3A + Automation Hat (already exists as edge-net-automation).

The Automation Hat has relay outputs capable of switching mains or low-voltage loads. The node already handles graceful shutdown of other nodes via SSH. The relay outputs are the interesting expansion.

**Open questions**:
- What does it switch? (lamp, fan, power socket, door strike?)
- What triggers it? (MQTT message from keybow, control panel, Telegram bot, schedule?)

---

## Household Task Tracker

Not a node on its own — an application that runs across several nodes.

**Concept**: Physical buttons log household tasks (e.g. "bed sheets changed — room 1"). Timestamp written to Postgres. A Telegram bot monitors intervals and messages when something hasn't been done in too long.

**Physical interface**: A button press on an Edge-NET node (Keybow, control panel, or a dedicated device) publishes an MQTT event. A service on the home network picks it up and writes to the database. No screen required to log the event.

**Ambient feedback**: The Kindle display could show task state. An LED color could indicate "all good" vs "something overdue."

**Telegram integration**: Bot polls the DB, surfaces only what needs attention. Two-way — bot can also receive "done" confirmations via chat if you're not near a physical button.

---

## AI Intent Engine — Dynamic Behavior Layer

The idea: a service on the home network uses a local LLM to decide what the nodes should be doing, rather than hardcoding that at design time. The open questions on nearly every node above ("what does the arrow point at?", "what does the Kindle display?", "what are the button mappings?") are all candidates for this layer to own.

### The boundary

**MQTT is the boundary between rigid and dynamic.**

Everything below it stays reliable and deterministic — how a button press becomes a publish, how a servo responds to a message. Everything above it is where the AI layer lives. Nodes never need to know an LLM is involved.

### Two levels of AI control

#### Level 1 — Config via MQTT (no redeployment)

Nodes run fixed firmware but subscribe to a config topic. The intent engine publishes:

```json
{
  "gnome_arrow": { "angle": 45, "meaning": "task overdue: kitchen" },
  "kindle_layout": "task_summary",
  "keybow_buttons": [
    { "button": 0, "action": "log_task:sheets_room1", "led": [255, 0, 0] },
    { "button": 1, "action": "log_task:hoover", "led": [0, 255, 0] }
  ]
}
```

The AI fills in the values; the schema is fixed. No USB, no SCP, no reboot. This covers most dynamic behavior — what a button means right now, what the display should say, where the arrow points.

#### Level 2 — Code generation + deploy

For fundamentally different hardware behavior (e.g. "this Pico now reads a temperature sensor instead of handling buttons"), the AI generates a `.py` file. A deployer service on the connected Pi SCPs it to the Pico, the Pico reboots, serial output is captured. If there's an error, it's sent back to the AI to fix. Loop until it works.

Safe version: the AI can only modify a `# USER LOGIC` block within a fixed template — the WiFi/MQTT setup header is untouchable.

### The intent engine service

A Python service on a Linux node (Pi 3A or a VM on pve1):

```text
[routine timer / MQTT events / sensor readings]
              ↓
       intent engine (Python service)
              ↓  assembles state packet
       Ollama via LiteLLM (Gaming PC / Mac Mini fallback)
              ↓  returns JSON config or .py script
       MQTT publish  /  SCP to Pico
```

It assembles a state packet — task tracker overdue items, sensor readings, node health, time of day — and sends that to the LLM with a tight output schema. The LLM decides what matters and what the nodes should reflect. Calls happen on a routine (every few minutes) or on significant state change, not on button press.

### What the AI owns vs. what stays hardcoded

| AI decides | Stays rigid |
| --- | --- |
| What a button means right now | That a button press publishes to MQTT |
| What the Kindle displays | How the Kindle polls and renders a PNG |
| Where the gnome arrow points | That the servo responds to its topic |
| What LEDs show | The MQTT topic schema |
| Generated `main.py` user logic block | WiFi/MQTT setup in the template |

### Open questions

- What goes in the state packet? (task tracker + sensor readings + node health + time of day is a start)
- Which node runs the intent engine? (Pi 3A is already `edge-net-automation`, a VM on pve1 is cleaner)
- How often does it call the LLM? (routine interval vs. event-triggered vs. both)
- For code gen: which nodes are candidates? (Pico W nodes only — Linux nodes can be reconfigured without redeployment)

---

## Software Deployment

Open problem worth solving properly. With AI-generated device code, the deployment pipeline becomes the bottleneck.

- **Linux/OpenBSD nodes** (hub, Pi Zero W, Pi 3A, Kindle): Ansible
- **CircuitPython/MicroPython nodes** (Pico W, Plasma Stick): auto-detect USB mount, sync repo directory

Goal: describe a change, generate the code, push to every relevant device in one command. This is where local LLM inference on the home network becomes genuinely useful — generate device code offline, deploy immediately.

### Three levels of update — prefer the lowest

The aim is to *avoid* reflashing. Most changes shouldn't touch a device at all.

1. **Data only (MQTT message).** Firmware fixed; the message changes *what* is shown, not *how*. e.g. Plasma — "set red". Never reflash.
2. **Declarative / config push.** Firmware is a generic interpreter; push a spec (JSON over MQTT), not code. **This is the target shape for GFX** — a dumb renderer that knows primitives (bar / line / sparkline / menu / text); the home network sends what to draw. New graph → new JSON → zero reflash. Same pattern as the Kindle (off-device PNG) and ideally the Inky.
3. **Code push (real OTA).** Push actual code, reflash. Heaviest, brick risk. Reserve for adding a *new primitive*.

This maps onto the AI levels above: Level 1 config-via-MQTT is level 2 here; Level 2 code-gen is level 3 here.

**Pico W can do true OTA over its own WiFi** — confirmed: MicroPython `umqtt` receives the file, `open('main.py','w')` writes flash, `machine.reset()` reboots. Not exotic. The USB cable to a Pi is the **unbrick backup** path (CircuitPython = USB mass-storage `cp`; MicroPython = `mpremote cp ... :main.py`), so keep a node's native micro-USB accessible. Brick guard: a tiny safe `main.py` that pulls `app.py`, falling back to last-known-good on crash.

### Surface roles by refresh rate

Displays differ by how fast they redraw — match the data's cadence to the surface:

| Changes every… | Surface | Role |
| --- | --- | --- |
| seconds | GFX LCD | live motion — graphs, menus, countdowns (the *verb*) |
| minutes | LEDs / Plasma | ambient colour shift |
| hours+ | e-ink (Kindle, Inky) | calm static state (the *noun*) |

The same metric can appear on all three at different cadences (e.g. energy: Plasma brightness = now, GFX = last-hour sparkline, e-ink = today's total). e-ink is the noun; GFX is the verb.

---

## What Can Connect

Things that make sense as Edge-NET nodes given the MQTT + WiFi substrate:

| Sensor / actuator | Interface | Notes |
|-------------------|-----------|-------|
| Temperature / humidity | I2C (BME280, SHT31) | One line of CircuitPython |
| Door / window open | GPIO (reed switch) | One pin, one resistor |
| Motion | GPIO (PIR sensor) | Same |
| Light level | I2C (BH1750) or ADC | |
| Sound level | ADC (microphone breakout) | |
| Relay outputs | Already have (Automation Hat) | Switches mains or low-voltage |
| RGB LED strip | Already have (Plasma Stick) | |
| E-ink display | Already have (Kindle) | |
| Servo | PWM (SG90) | Already have (gnome) |

The constraint is never the sensor — it's deciding what to *do* with the data.
