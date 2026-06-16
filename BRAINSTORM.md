# Edge-NET — Brainstorm Brief

A self-contained brief. Paste the whole thing into a fresh LLM chat and ask for
ideas. It assumes no prior context. The structured source of truth is
[capabilities.yaml](capabilities.yaml); this is the readable version for a human
or an LLM to reason over.

---

## The ask

I have a small physical network of devices in my home (details below). I want
**lots of different ideas** for what to build with it. Be expansive and varied —
practical, playful, weird, ambitious. For each idea, consider:

- **Which devices it uses** (inputs and outputs from the inventory).
- **Where each device physically lives** in a house (which room, what surface).
- **How a person interacts with it without reaching for a phone or a screen** —
  this is the whole point. The room should express what's going on; you log and
  read things through physical buttons, lights, an arrow, an e-ink panel.

A guiding example: *how do I record that I've put the bedsheets in the wash, and
get nagged if it hasn't happened in two weeks — without opening an app?*

Don't converge on one answer. Give me a long, branching list.

---

## What Edge-NET is

A self-contained WiFi network built from Raspberry Pi hardware. A hub (Pi 4
running OpenBSD) creates a WiFi access point and runs an MQTT broker. Every other
device connects to that WiFi and talks over MQTT — publishing events, subscribing
to commands. No internet required. Plug in an ethernet cable and extra resources
on my home network unlock (a database, local LLMs, Grafana).

**MQTT is the boundary.** Below it: rigid, reliable device behaviour (a button
press becomes a message; a servo moves on a message). Above it: anything can
react to anything. A button on one device can light an LED, move an arrow, switch
a relay, or write a row to a database — they never need to know about each other.

---

## The devices (inputs and outputs)

### Hub — Raspberry Pi 4, OpenBSD
The centre. WiFi access point, firewall, DHCP, MQTT broker. No IO of its own.

### Keybow — Pi Zero W + 3-key pad  *(built)*
- **3 physical buttons**, each publishes a press.
- **3 RGB LEDs**, one behind each button, network-controllable.
- Two-way: light a button to say "this needs doing", press it to confirm done.
- Runs full Linux, can also write to the database directly.

### Gamepad — Adafruit Mini I2C Gamepad on a Plasma Stick  *(scaffold)*
- **6 buttons** (A, B, X, Y, Start, Select) — press/release.
- **Analog joystick**, 2 axes (continuous, 0–1023).
- The richest input on the network — a natural universal remote / menu navigator.
- The same board also drives a **50-LED RGB strip** (see below).

### Plasma — 50-LED RGB strip  *(on the gamepad board)*
- **50 addressable RGB LEDs.** Solid colour, patterns, brightness, per-pixel.
- Can be folded into a **10×5 pixel grid** to scroll short text.
- Pure ambient output. (A second spare strip exists if a standalone one is wanted.)

### GFX — Pi Pico W + small graphical display  *(stub)*
- **128×64 monochrome LCD** — text, graphics, graphs, sparklines, menus.
- **RGB backlight** — colour as an at-a-glance signal.
- **5 buttons** (4 for the current app, 1 for back/home).
- The interactive terminal of the network. Can read live data from Grafana.

### Automation — Pi 3A + Automation Hat  *(stub)*
- **3 relays** (24V @ 2A) — switch a lamp, fan, mains socket, door latch.
- **4 ADC inputs** + **3 buffered inputs** — read sensors, dials, a reed switch
  on a cupboard or door.
- **3 sinking outputs**, **15 indicator LEDs**.
- Also orchestrates graceful power-off of other nodes (SSH then cut the relay).

### Kindle — jailbroken Kindle 4, e-ink  *(scaffold)*
- **600×800 grayscale e-ink display.** Renders a PNG generated on the home
  network. Calm by default; escalates only when something needs attention.
- **D-pad + page-turn buttons** available as inputs (currently unused).
- Ambient hallway/desk display — weather, task state, node health.

---

## Interaction primitives (the raw vocabulary)

- **Discrete input:** 3 Keybow buttons, 6 gamepad buttons, 5 GFX buttons, Kindle
  d-pad/page buttons, 3 Automation buffered inputs (reed switches etc.).
- **Continuous input:** gamepad joystick (2 axes), 4 Automation ADC channels.
- **Ambient output:** 50-LED strip, 3 Keybow RGB LEDs, GFX backlight colour,
  15 Automation indicator LEDs.
- **Rich output:** GFX 128×64 LCD (graphs/menus), Kindle 600×800 e-ink.
- **Physical actuation:** 3 relays — switch real-world power.
- **Data sinks:** Postgres on the home network; Grafana/Prometheus.

---

## External resources (optional, home network)

Edge-NET runs standalone without these. With the ethernet uplink they unlock a
smarter, persistent layer:

- **Database** — PostgreSQL + pgvector + Apache AGE (graph) + Redis. Persist
  events, timestamps, state, vectors, graph relations. (The bedsheets log lives
  here.)
- **Local + cloud LLMs** via a router (LiteLLM → local Ollama on a GPU box and a
  Mac, plus Anthropic/OpenAI). Could power an "intent engine" that decides what
  the nodes should reflect, or generates device behaviour on the fly.
- **Grafana + Prometheus** — metrics the GFX display can read back.

---

## Constraints and taste

- **No phone, no app as the primary interface.** Physical-first. The environment
  expresses state; you act through buttons and read through light/e-ink/arrows.
- **Ambient over alerting.** Calm by default, escalate only when it matters.
- **Decoupled.** Everything goes through MQTT — assume any input can drive any
  output.
- **It's a home.** Rooms, hallways, kitchen, bedroom, desk. Real daily routines
  (chores, presence, weather, comings and goings) are fair game.

Now — give me lots of ideas.
