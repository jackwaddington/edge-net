# Edge-NET — Principles

Edge-NET is the **interactive part of the house** — an IoT fabric where the
environment expresses state and you act through physical things. These principles
say *how* it's built, and guard against the obvious wrong turns.

---

## 1. Capability over device

A use-case is a concern of the **whole system**, not a job pinned to one box.
"Observability", "chores", "presence" are expressed *across* the network — many
inputs, many outputs — not embodied in a single device.

> Wrong: "the GFX screen *is* the observability thing."
> Right: observability is a concern; the GFX screen is one surface that can
> reflect it, alongside LEDs, e-ink, an arrow, a relay.

This is why `edge-display` was folded into a node: it was a use-case wearing a
device's name.

## 2. Devices are IO primitives, not apps

Each node exposes **raw inputs and outputs** — buttons, LEDs, a screen, a relay,
an ADC. It does not own a purpose. A button is "a button that publishes a press",
not "the bedsheets button". Purpose is assigned above the bus and can change
without touching the device.

## 3. One device, one job is the anti-pattern

The instinct to make each device do a single fixed thing is the thing to resist.
It is not how an interconnected world works. Any input may drive any output;
the same hardware serves many concerns over its life.

## 4. MQTT is the boundary

Below the bus: rigid, reliable, dumb device behaviour — a press becomes a
message, a message moves a servo. Above the bus: anything reacts to anything.
The contract is the topic, not the wiring.

## 5. Decoupled — any input → any output

Devices never know about each other. A Keybow press can light a Plasma strip,
move an arrow, switch a relay, or write a database row, and none of them are
aware of the others. Rewiring behaviour is rewiring subscriptions, not firmware.

## 6. Dumb nodes, smart edge

Keep display logic and business logic **off** the constrained devices,
especially the microcontrollers. A node receives data or a small spec and
renders it; it should rarely need reflashing. The thinking lives on the home
network (DB, LLM, rules) above the bus.

## 7. Ambient over alerting

Calm by default. The house reflects state quietly — colour, a glanceable panel —
and escalates only when something genuinely needs attention.

## 8. Physical-first, no phone

The room is the interface. You log and read through buttons, light, an arrow,
e-ink. A phone or app is never the primary way in. This is the whole point.

## 9. Standalone, progressively enhanced

The network runs self-contained with no internet. An ethernet uplink *unlocks*
more — persistence (Postgres), intelligence (LLMs), metrics (Grafana) — but is
never required for the core to work.

---

**Test for any new idea:** does it express a *concern* across the fabric, or does
it make one device do one job? If the latter, it's probably wrong.
