<!-- title: Architecture -->
# Architecture

Agentic Codex runs AI agents as long-lived **Claude Code** sessions on a single Debian-based VPS. Three
layers, each with a clear owner and a clear trust level.

```
┌───────────────────────────────────────────────────────────────────────┐
│  GitHub org  <ORG>                                                      │
│   kb-agent-shared   infra   kb-agent-<role>-<name> (one per agent)      │
│   (governance)      (host)  (brains: identity, memory, skills, tools)   │
└───────────────────────────────────────────────────────────────────────┘
                    │ git (portable source of truth)
                    ▼
┌───────────────────────────────────────────────────────────────────────┐
│  Debian-based VPS  <VPS_HOST>                                           │
│                                                                         │
│   Host layer (root):  claude-topic wrapper · kb-sync · agentic-monitor  │
│                                                                         │
│   root-agent (sudo)      cos-agent           dev-agent      …           │
│   Unix user + brain      Unix user + brain    Unix user + brain         │
│      │                       │                    │                     │
│   systemd user units: claude --remote-control (one per "topic")         │
└───────────────────────────────────────────────────────────────────────┘
                    ▲ Remote Control (web / mobile / desktop)
                    │
              Your devices
```

## The three layers

### 1. Brains — one Git repo per agent

Each agent has a repo `kb-agent-<role>-<name>` containing its identity and durable memory as plain
Markdown ([OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog)-inspired):

- `SOUL.md + OPERATING.md` — the canonical identity and operating contract (framework-agnostic).
- `CLAUDE.md`, `AGENTS.md` — thin per-framework adapters that point to them.
- `memory/`, `skills/`, `tools/`, `context/` — durable memory, self-authored procedures, tool notes.
- `deploy/` — the runtime wiring (`topics.tsv`, `home-CLAUDE.md`, `claude-settings.json`).
- `shared/` — a symlink to the sibling clone of `kb-agent-shared`.

Because a brain is just Markdown in Git, you can read it, diff it, edit it from your phone, roll it
back, and move it to another box or another framework. See [`portability.md`](portability.md).

### 2. Governance — one shared repo

`kb-agent-shared` holds what must be **identical across every agent**: global policies (approval,
memory, source-of-truth, access), OKF templates, cross-agent decision records, runbooks, and a
cross-agent handoff channel. Every brain reaches it through a committed symlink
`shared -> ../kb-agent-shared` to a **sibling clone** (not a git submodule — no pinning, no
detached-HEAD, always-latest via `kb-sync`).

### 3. Host layer — one infra repo, installed with root

`infra` carries everything that needs root or is shared by all agents:

- **`claude-topic`** — the wrapper that supervises Remote-Control sessions ("topics"). Installed
  **root-owned** at `/usr/local/bin/claude-topic` so no agent can rewrite what another agent executes.
- **`claude-topic@.service`** — a per-agent systemd **user** unit that keeps one
  `claude --remote-control` process alive (crash + reboot survivable, no tmux).
- **`kb-sync`** — a 15-minute timer that fast-forwards every agent's Git clones (inert data only).
- **`agentic-monitor`** — a 5-minute host timer that heartbeats an external dead-man's switch.
- **`provision-agent`** / **`install-host-services`** — the explicit bring-up scripts.

See [`runtime.md`](runtime.md) and [`monitoring.md`](monitoring.md).

## Agents: lanes, not sandboxes

You can run a single agent. When you run several, they are separated by **role and Unix user**, and
exactly one — the **root-agent** — is privileged (has `sudo`, owns the host). The others are
unprivileged frontends in their own lanes (e.g. a broad, unprivileged chief-of-staff; a build/deploy
agent with its own scoped deploy token and never root).

Be honest about what this is: on one box, "separate agents" is a **governance** model (who initiates
what work), **not** a hard security boundary — the privileged agent can operate on the others' files as
their Unix user for maintenance. The security posture that matters is: **one** privileged principal,
kept off broad content-ingesting integrations; everyone else unprivileged. See
[`multi-agent-governance.md`](multi-agent-governance.md).

## Why Claude Code, why a Debian base

- **Claude Code + Remote Control** is the happy path because it surfaces each long-lived session on
  web, iOS/Android, and desktop at once — you talk to your agents from anywhere, no extra gateway to
  run and patch. The brains themselves are framework-agnostic ([`portability.md`](portability.md)), so
  this is a runtime choice, not a lock-in.
- **A Debian-based distro** because the resilience model is plain **systemd** (user services + timers + linger) — no
  bespoke supervisor.

## The property that ties it together

Everything important lives in Git; the box holds only secrets and volatile runtime state. That single
property is what makes the system portable *and* recoverable *and* auditable — and it's enforced by the
three-tier config model in [`config-model.md`](config-model.md), which is the most important document
here after this one.
