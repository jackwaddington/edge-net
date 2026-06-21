# The cut-the-cable demo

The one thing Edge-NET has to prove: **a fully functioning system that keeps
working with the internet cut off.** Everything below serves that single gesture.

## The thesis (already true, needs *showing*)

Nothing in the data plane touches the internet. The hub (OpenBSD Pi) makes its
own WiFi AP (`Pirie`), its own DHCP (10.1.1.x), and its own MQTT broker
(10.1.1.1). Nodes talk through that broker, locally. The home ethernet uplink
(`bse0`) is *optional* — only for extras (Grafana). So the demo is not to build
independence, but to **demonstrate** it.

## The gesture

1. Power the **hub** (AP + broker).
2. Every node on power, joined to `Pirie`: gamepad (in), GFX (in+out),
   Inky (in+out), keybow (in), automation/relays (out).
3. Press inputs → outputs react: gamepad → GFX flash + Inky + LED ripple;
   keybow → relay clunk + LED.
4. **Pull the hub's ethernet / kill all internet.** Press again. Nothing changes.
   No cloud, no router, no internet — buttons still drive relays and displays.

That moment is the whole pitch.

## What's working (2026-06-18)

- **keybow → relay** round-trip (via a rule on the Pi 3A): button → relay clunk +
  LED feedback. Solid, great demo material — the *clunk* sells it.
- **gamepad → GFX**: live, sub-second, flashes the pressed button big + colour.
- **gamepad → Inky**: live; slow e-ink "last button" board (the calm noun).

## The gap: outputs must look ALIVE (feedback 2026-06-18)

A demo dies if the outputs look static. Every output node needs **ambient
liveness + reaction**, not a dead screen or a grey blip:

- **LED strips** (gamepad's own 50-LED strip; the weather Plasma stick): today the
  gamepad strip only does a dim grey blip on press — too weak. Wanted:
  - **Ambient:** a gentle idle animation (slow breathing glow / soft colour drift)
    that says "I'm online" at a glance.
  - **Interrupt:** on a command/button, a **ripple** or chase down the strip.
  This is the ambient+interrupt pattern in light. It also doubles as a liveness
  indicator — you can *see* a node is connected without a screen.
- **Inky:** beyond "last button," show node health / "online" as the calm status.
- **GFX:** already lively (the verb); fine.

## Progress (2026-06)

- ✅ **GFX + Inky buttons publish** — every display node is now two-way; the mesh
  is real (any button anywhere can drive any output).
- ✅ **Generalised rule** on the Pi 3A — any input node's buttons toggle relays
  *and* flash the Plasma strip a colour. Deployed live over the network.
- ✅ **LED framebuffer** (edge-net-plasma) — the Plasma strip is now a **dumb
  framebuffer display**: receives hex pixel frames over MQTT, blits them, smooth
  playback. All rendering (text, 5×10 mapping, animation) lives off-device in a
  sender. Addressing + smooth flowing rainbow verified. This *is* the "make it
  look alive" piece, done the clean way.
- ⬜ Still wanted on top: an idle ambient glow + ripple (a sender behaviour now,
  not firmware), and folding the strip into a 5×10 matrix for scrolling text.

## Out of scope for this demo

**OTA** is a different axis entirely — it's about updating code without a USB
cable, i.e. maintainability. The running system needs no internet with or without
OTA. Don't let OTA block the demo. (Plan: prove OTA on the easy-to-reach GFX/Inky
first — see `edge-net-ota-strategy` in memory and PLANNING.md "Software Deployment".)

## Next-session checklist

- [x] GFX/Inky buttons publish (two-way mesh).
- [x] Rule generalised: any button → relays + strip colour.
- [x] LED framebuffer: smooth addressable strip display.
- [ ] **GFX-as-handheld**: read the GamepadQT (now on the GFX) → games + a
  letter-picker that streams text to the LED matrix.
- [ ] Fold the strip into a 5×10 serpentine matrix; verify `xy_to_index`.
- [ ] WiFi fallback (`Pirie` → home WiFi) + broker on the hub's home side.
- [ ] Power each node, confirm it joins `Pirie` and appears on the broker.
- [ ] Stage the full press → react chain; dry-run the cable pull (zero change).
