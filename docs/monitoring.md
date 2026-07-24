<!-- title: Monitoring — a dead-man's switch, because silence is the alarm -->
# Monitoring

One small, robust monitoring component — a host-level health check that heartbeats an **external**
dead-man's switch. The design principle: **silence is the alarm.** If the box, Docker, the network, or
the monitor itself dies, you still get told, because the *absence* of a heartbeat is what triggers the
alert.

## What runs

**`agentic-monitor`** runs on the host (OUTSIDE Docker, so it survives a Docker/Dokploy failure), every
~5 minutes via a systemd timer. Each run does a few binary local checks, then pings
[healthchecks.io](https://healthchecks.io):

- all ok → ping `<HC_URL>` (heartbeat)
- problem, 1st cycle → ping `<HC_URL>` with a "pending" body (no alarm yet — hysteresis)
- problem, 2nd consecutive cycle → ping `<HC_URL>/fail` with the detail → healthchecks alerts you
- script/timer/box/network dead → no ping at all → healthchecks alerts on the missed heartbeat

Checks include: Docker daemon up and no unhealthy/restarting containers; **critical containers present**
(catches an exited/vanished reverse proxy) and **HTTP liveness** (catches "up but broken"); disk usage;
available memory; failed systemd units; `kb-sync` freshness; and **every agent's topic sessions active**.

Two properties keep it trustworthy:
- **Hysteresis** — a problem must persist **2 cycles** before a `/fail`, so a single blip doesn't page you.
- **Fail-safe** — the alert path lives *off* the box (healthchecks.io → Telegram/your channel), so it
  works precisely when the box doesn't.

## Setup (done during bring-up)

1. `sudo ./scripts/install-host-services` installs the `agentic-monitor` timer (plus `kb-sync` and the
   weekly `agentic-update-check`).
2. Create a check on healthchecks.io — **period ~5 min, grace ~20 min** — and connect it to **Telegram**
   (or email/Slack/etc.).
3. Put its ping URL into `/etc/agentic-monitor.env` (mode 600, **not in Git**) as `HC_URL`.
   Optionally add a second check as `UPDATE_HC_URL` for the weekly update report.

```ini
# /etc/agentic-monitor.env   (mode 600 — the ping URL is a capability, keep it secret)
HC_URL=<HC_URL>
DISK_MAX=90
MONITOR_EXPECT_CONTAINERS="traefik dokploy"     # if you run Dokploy
MONITOR_HTTP_CHECKS="traefik=http://localhost:80 dokploy=http://localhost:3000"
```

## Operating it

- **Test detection, no ping:** `sudo agentic-monitor --dry-run`.
- **Test the alert chain:** `curl --data-raw "test" "$HC_URL/fail"` then `curl "$HC_URL"` (clears).
- **When it fires:** read the Telegram body — on a `/fail` it names what broke
  (`containers:…`, `disk:92%`, `topics-down[<agent>]:…`, `kb-sync:stale(50m)`, `failed-units:…`). A
  **missed heartbeat** (no body) means the box/Docker/network/monitor is down — inspect the host
  directly.
- **Silence known noise:** add substrings to `MONITOR_EXCLUDE_CONTAINERS` / `MONITOR_EXCLUDE_UNITS` so a
  real problem stands out. Keep excludes few and documented.

## Portability

The monitor is part of the `infra` repo, so a new box gets it from `install-host-services`; only the
secret `HC_URL` is restored out-of-band. Same story as everything else here — the config is in Git, the
secret isn't.

## Deliberately not covered

- **Backup restore-tests** — a good second check to add (a healthchecks.io check that a backup job
  actually ran), but out of scope for the base template.
- **App deploy events** — Dokploy's own notifications can cover these; they run in Docker, so they're
  not on the critical alarm path (which must survive Docker being down).
