# ADR 0001 — Deployment tooling: systemd + git, not Ansible or balena

**Status:** accepted · 2026-06-17

## Context

How should node software be deployed and kept running on Edge-NET? Candidates
considered: Ansible, balena.io, plain Docker, Terraform, and "systemd + git".

The decision is shaped by the shape of the fleet, which is small and very
heterogeneous:

- **Hub** — Pi 4 running **OpenBSD** (pf, hostap, dhcpd, Mosquitto).
- **Linux SBCs** — keybow (Pi Zero W), automation (Pi 3A). Only these two.
- **Microcontrollers** — GFX, Plasma, gamepad, Inky (Pico W / RP2040). No OS.
- **Kindle** — jailbroken Kindle 4 (its own ancient Linux).

And by [PRINCIPLES.md](../PRINCIPLES.md): the network is **self-contained, no
internet required**; the uplink only *progressively enhances*.

## Decision

Use **systemd + git** as the **default** for the Linux nodes: code in a per-node
repo, pulled/copied to the node, run as a `systemd` service with `Restart=always`
(auto-start on boot, self-heal on crash/blip).

**balena is a legitimate option for the *capable* Linux nodes (the Pi 3A) as an
ergonomics upgrade** — git-push deploy, declared dependencies, restart policies,
and a remote terminal that sidesteps jump-host/NAT pain. Adopt it there if the
manual provisioning friction (scp drift, apt/pip, sudo/group setup) is judged
worth the **cloud control-plane dependency** (or self-hosted openBalena). Do
**not** put it on the **Pi Zero W** (ARMv6, 512MB — balena's worst-case
hardware), and note it **does not address link reliability** (the dominant
failure mode here was the AP channel, not the deploy method).

**Reject Ansible and Terraform** as the primary mechanism.

## Rationale

- **balena** genuinely solves the *provisioning ergonomics* that hurt during
  bring-up (manual scp, code drift between repo and node, apt/pip/PEP-668,
  sudo/user/group setup, remote access through NAT). The scope is the **Linux
  nodes only** — the OpenBSD hub can't run it and the microcontrollers/Kindle are
  out, but that's a clean, valid split, not a reason to dismiss it. Caveats: it
  adds a **cloud control plane** (balenaCloud; openBalena self-hosts but is
  heavier), which is in mild tension with the self-contained principle; on a
  **Pi Zero W (512MB, ARMv6)** the container runtime is mostly overhead (poor
  ARMv6 image support, daemon eats scarce RAM/CPU); and it **does not fix link
  reliability** — the real bring-up blocker was the AP defaulting to congested
  2.4GHz channel 1 (fixed by pinning channel 11), which no deploy tool addresses.
  Net: reasonable on the Pi 3A as an ergonomics choice, not on the Pi Zero W.
- **Ansible** — heavier than this scale needs; the team is not standardising on
  it. The repos already carried half-wired Ansible that never ran cleanly.
- **Terraform** provisions infrastructure (VMs, cloud, DNS), not edge-app
  deployment — not relevant here.
- **systemd + git** is featherweight, runs on every Linux node including the Pi
  Zero, needs no cloud, and `Restart=always` gives the boot/blip resilience that
  matters most (the Pi Zero's WiFi is flaky — see node notes).

## Consequences

- Microcontrollers are out of scope for all of the above — they are flashed over
  USB / updated by their own mechanism (see the update model in PLANNING.md).
- Each Linux node owns a `systemd/` unit in its repo (`Restart=always`, run as
  the `edge` service user).
- Revisit balena only if Edge-NET ever becomes *many identical Linux Pis*.
