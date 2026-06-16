# Edge-NET — Use Cases

What the system needs to *do*. Distinct from [capabilities.yaml](capabilities.yaml)
(what the hardware *can* do) and [BRAINSTORM.md](BRAINSTORM.md) (open-ended idea
generation). This is the decided backlog — things worth building.

Each use case lists the **devices** it touches and the **software service** that
implements it. Services live in the `edge-net-apps` monorepo (above the MQTT line).
Promote a use case to issues when you commit to building it.

Status: `idea` → `speccing` → `building` → `done`.

---

## UC-1 — Household task tracker ("days since")

**Need:** Log a recurring chore (e.g. bedsheets changed) with one physical press,
no phone. See how many days since / until due. Get nagged if overdue.

- **Input:** a Keybow button (or gamepad button) → MQTT event.
- **Persist:** Postgres — `(task, timestamp)` rows; intervals per task.
- **Ambient output:** Kindle shows "sheets: 11 days" / Keybow LED goes red when
  overdue / LED strip colour.
- **Nag:** Telegram bot messages when an interval is exceeded; can also accept a
  "done" reply.
- **Service:** `task_tracker` (subscribes to button topics, writes DB, computes
  due state, publishes display + LED state, runs Telegram bot).
- **Status:** idea

## UC-2 — Timelapse camera monitor

**Need:** Know the Raspberry Pi Zero timelapse camera is alive and see its latest
frame, at a glance, without opening a laptop.

- **Source:** the Pi Zero timelapse camera (separate device — its own firmware repo).
- **Display:** GFX shows status + latest frame thumbnail; Kindle could show the
  latest full frame.
- **Service:** `timelapse` (pulls latest frame / health, republishes to GFX topic
  or generates a Kindle PNG panel).
- **Status:** idea
- **Open:** does the camera already publish anywhere, or do we poll its filesystem?

## UC-3 — Infrastructure monitoring (Prometheus / Grafana)

**Need:** See home infra health (k3s cluster, nodes, services) on a physical
display — the GFX terminal or the Kindle — rather than logging into Grafana.

- **Source:** Grafana/Prometheus on k3s (192.168.0.216:3000/9090). GFX is already
  firewalled through to it.
- **Display:** GFX app for cluster health; Kindle panel for a calm summary.
- **Service:** `monitoring` (queries Prometheus, formats for GFX topic / Kindle PNG).
- **Status:** idea

## UC-4 — Kindle ambient dashboard

**Need:** A calm e-ink panel that shows the right things (task due-states, weather,
node/infra health) and escalates only when something needs attention.

- **Display:** Kindle (polls an HTTP PNG).
- **Service:** `kindle_dashboard` (generates the PNG from DB + other services'
  state, serves over HTTP). This is the aggregator several use cases feed into.
- **Status:** idea

---

## Cross-cutting: the intent engine (later)

A service that uses the local LLM (via LiteLLM) to decide what nodes should
reflect, rather than hardcoding it. Sits above all of the above. See
[PLANNING.md](PLANNING.md). Status: idea, deferred.

---

## Backlog (unsorted, from brainstorm)

_Add ideas here as they land. Promote to a UC above when serious._
