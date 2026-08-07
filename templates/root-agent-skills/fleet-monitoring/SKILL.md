---
name: fleet-monitoring
type: skill
title: Monitoring & Alerting (agentic-monitor + healthchecks.io)
description: Operational runbook for the fleet's single monitoring component — the host-level agentic-monitor check and its external dead-man's switch — how to read an alert, test it, tune noise, and reinstall on a new box.
tags:
- skill
- monitoring
- alerting
- infra
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Monitoring & Alerting

Design + rationale: the Agentic Codex docs (`monitoring.md`). This is the operational side.

## What runs

- **`agentic-monitor`** on the host (OUTSIDE Docker), every 5 min via `agentic-monitor.timer`, pings
  healthchecks.io. Checks: docker daemon up + no unhealthy/restarting containers; **critical containers
  present** (dokploy/traefik core — catches an exited/vanished proxy) + **HTTP liveness** of traefik/dokploy
  (catches an up-but-broken reverse proxy = site down); disk; **available memory**; failed systemd units;
  kb-sync freshness; **every agent's topic sessions active**.
- **Hysteresis:** a problem must persist **2 cycles** before a `/fail` (no single-blip alarms).
- **Config:** `/etc/agentic-monitor.env` (mode 600, **not in git**) — `HC_URL`, `DISK_MAX`,
  `MONITOR_EXCLUDE_CONTAINERS`, `MONITOR_EXCLUDE_UNITS`.
- **Source (portable):** `<ORG>/infra` — `scripts/agentic-monitor`,
  `systemd/agentic-monitor.{service,timer}`, `systemd/agentic-monitor.env.example`.

## healthchecks.io

- Check `<VPS_HOST>`, **period 5 min, grace ~20 min**, delivery = **Telegram**.
- Two alarm triggers: a **`/fail`** ping (a check failed — detail in the body) **or** a **missed
  heartbeat** (script/timer/Docker/host/network dead). **Silence is the alarm.**

## When an alert fires

1. Read the Telegram body. On a `/fail` it names what broke, e.g. `containers:foo`, `disk:92%`,
   `topics-down[cos-agent]:ops`, `kb-sync:stale(50m)`, `failed-units:bar.service`.
2. If it's a **missed heartbeat** (no body / "went down"), the box, Docker, network, or the monitor
   itself is down — inspect the host directly (Cockpit on `:9090` as a generic option, SSH via Tailscale).
3. Fix the cause; the next clean cycle pings success and healthchecks clears automatically.

## Common ops

- **Test detection (no ping):** `sudo agentic-monitor --dry-run`.
- **Test the Telegram chain:** `curl --data-raw "test" "$HC_URL/fail"` then `curl "$HC_URL"` (clears).
- **Silence known noise:** add a substring to `MONITOR_EXCLUDE_CONTAINERS` / `MONITOR_EXCLUDE_UNITS` in
  `/etc/agentic-monitor.env` — exclude the known so a real problem stands out. Keep excludes few and
  documented.
- **Add a check:** append to the `problems` array in `scripts/agentic-monitor`, commit and push it,
  then reinstall with `install-host-services`. Not by hand: a direct
  `sudo install … /usr/local/bin/agentic-monitor` puts unreviewed code in root's path, writes no
  manifest row, and leaves the daily divergence check comparing the new binary against whatever
  source the manifest still names.

## Portability (new VPS)

Clone `<ORG>/infra`, run `scripts/install-host-services` (installs kb-sync + agentic-monitor +
timers), then set `/etc/agentic-monitor.env` (`HC_URL` from your healthchecks.io check). See
the Agentic Codex docs (`config-model.md`).

## Not covered (by choice)

- **Backup restore-test** — deferred (the owner trusts your VPS provider for now); a future second
  healthchecks.io check can cover "backup job didn't run".
- **App deploy events** — Dokploy's native notifications can cover these if wired; not load-bearing, and
  they run in Docker so they are not the critical alarm path.
