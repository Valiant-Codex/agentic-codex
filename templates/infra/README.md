# infra — the host layer

This is the **host layer** template: everything that needs root on the VPS and is shared by every
agent. It is one of the repos your root agent creates in your GitHub org (`<ORG>/infra`) from this
template. Nothing here contains a secret — real secrets live on the box, out of git (see
`docs/secrets.md`).

## What's in here

| Path | What it is |
|---|---|
| `bin/claude-topic` | The control surface for an agent's Remote-Control sessions. Installed **root-owned** at `/usr/local/bin/claude-topic`; drives the systemd units below. See `docs/runtime.md`. |
| `systemd/claude-topic@.service` | Per-agent **user** unit that supervises one `claude --remote-control` session (survives crash + reboot, no tmux). |
| `systemd/kb-sync.{service,timer}` | Host timer that fast-forwards every agent's git clones every 15 min (Tier-2: inert data only). |
| `systemd/agentic-monitor.{service,timer,env.example}` | Host timer (~5 min) — health check + dead-man's-switch heartbeat to healthchecks.io. `docs/monitoring.md`. |
| `systemd/agentic-update-check.{service,timer}` | Weekly OS + Dokploy update report (informational). |
| `scripts/install-host-services` | Installs the three host timers above. Run once per box. |
| `scripts/provision-agent` | Tier-3: brings one agent fully up from git + its restored secret. The single bring-up / recovery path. |
| `scripts/kb-sync`, `scripts/agentic-monitor`, `scripts/agentic-update-check` | The scripts the timers run. |

## Bring-up order (on a fresh box)

```bash
# 1) host services (once)
sudo ./scripts/install-host-services
#    then set the real HC_URL secret:  sudo nano /etc/agentic-monitor.env   (mode 600)

# 2) per agent (once each) — ORG is your GitHub org that owns the repos
ORG=<your-github-org> ./scripts/provision-agent <unix-user> <brain-repo>
```

`provision-agent` is **idempotent** — re-run it any time to converge a box back to the git state.
Both scripts self-elevate (`sudo`) as needed.

## Placeholders to replace

`<ORG>` — your GitHub org. Agents and their repos are auto-discovered at runtime
(`~/github/<org>/kb-agent-<role>-<name>`), so most scripts need no editing; the few literal `<ORG>`
strings above are in comments/report text only.

## Why root-owned wrapper + explicit provisioning

The wrapper is the one thing every agent *executes*, so it is installed root-owned and is **only**
ever written by the explicit Tier-3 `provision-agent` step — never by the 15-minute auto-sync. That
split (git = portable source of truth; provisioning = apply + security boundary; kb-sync = refresh
inert data only) is the core of the model. Read `docs/config-model.md`.
