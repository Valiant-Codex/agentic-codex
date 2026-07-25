<!-- title: Agentic Codex -->
# Agentic Codex

**Run your own fleet of AI agents on a VPS — with brains you can read, own, and move.**
Framework-agnostic. Portable. Minimal attack surface. Reachable from web, mobile, and desktop.

*By [Valiant Codex](https://github.com/Valiant-Codex) · created by Dario Valiant Casilli. MIT licensed.*

---

## What this is

Agentic Codex is a **blueprint + ready-to-use templates** for running one or more AI agents as
long-lived [Claude Code](https://docs.claude.com/en/docs/claude-code) sessions on a plain Debian-based VPS,
where:

- **Each agent's "brain" is a Git repo** — its identity, memory, skills, and tools are plain Markdown
  you can read, diff, edit from your phone, and move to another machine or another agent framework seamlessly.
- **The runtime is supervised and self-healing** — sessions survive crashes and reboots (systemd, no
  tmux) and are reachable from any device via Claude Code **Remote Control** (web, iOS/Android, desktop).
- **Portability and security are one clean boundary** — Git is the portable source of truth; a single
  explicit provisioning step applies it to a live box; a 15-minute auto-sync only ever refreshes inert
  data. Moving to a new VPS is *clone + one command*.
- **The surface is small and yours** — no extra gateway to keep patched, no black-box memory layer,
  secrets never live in Git.

It is the write-up of a system that actually runs a real fleet of agents day to day. The reference
architecture that inspired it is described in
[`docs/reference-architecture.md`](docs/reference-architecture.md).

## Why it looks the way it does

These were the design drivers — if you share them, this repo is for you:

| Goal | How it's met |
|---|---|
| **Own your agents' memory** | Brains are Git repos of Markdown (an [OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog)-inspired structure), not a vendor's memory store. |
| **Framework portability** | Canonical identity lives in `SOUL.md + OPERATING.md`; thin adapters (`CLAUDE.md`, `AGENTS.md`) point to it, so the same brain works across agent runtimes. See [`docs/portability.md`](docs/portability.md). |
| **Talk to agents from anywhere** | Claude Code Remote Control surfaces each session on web/mobile/desktop — the reason this happy path is Claude Code. |
| **Minimal attack surface** | One privileged agent, unprivileged others; no extra always-on gateway; a root-owned wrapper no agent can rewrite. |
| **Recover / migrate in minutes** | `clone + provision-agent`. Nothing important lives only on the box. See [`docs/config-model.md`](docs/config-model.md). |
| **Fail loud, cheaply** | A host-level dead-man's-switch heartbeats an external service; silence is the alarm. See [`docs/monitoring.md`](docs/monitoring.md). |

## The mental model

```
   your devices - web / mobile / desktop
            |  Claude Code Remote Control
            v
   +------------------------------------------------------+
   |  Debian-based VPS  <VPS_HOST>                        |
   |                                                      |
   |   root-agent (sudo)  |  cos-agent  |  dev-agent      |
   |   one Unix user each - each runs:                    |
   |   claude --remote-control   (supervised by systemd)  |
   |                                                      |
   |   each brain = a Git repo (Markdown)                 |
   |   host layer: claude-topic, kb-sync, agentic-monitor |
   +------------------------------------------------------+
            |  git
            v
   GitHub org: brains + infra + governance
```

- **Host layer** (`templates/infra`) — everything that needs root: the `claude-topic` session
  supervisor, the `kb-sync` git auto-refresh, the `agentic-monitor` dead-man's-switch. Installed once.
- **Agents** — one Unix user each; one privileged **root-agent** (has sudo, owns the host), plus any
  number of unprivileged agents in their own lanes. One agent can be enough.
- **Brains** (`templates/kb-agent-template`) — one Git repo per agent, plus a shared governance repo
  (`templates/kb-agent-shared`) that every agent reads.

## What a human has to do (the whole list)

The design goal is that a person does **only** the irreducible setup, and the agent does the rest by
reading [`README_AGENT.md`](README_AGENT.md). **New to servers?** The
[getting-started guide](docs/getting-started.md) walks the steps below through click-by-click — renting
a VPS, reaching it via Tailscale, and locking it down so nothing is exposed. The human steps:

1. **Subscribe to Claude** (a plan that includes Claude Code).
2. **Rent a VPS (choose a debian-based distribution, like Ubuntu).**
3. **Create a Unix user** for your root agent, give it `sudo`, and enable *linger*
   (`sudo loginctl enable-linger <user>` — this keeps the agent's sessions running even when no one is
   logged in, so they survive reboots).
4. **Create a free GitHub organization.**
5. **Create a GitHub account** (a bot/machine user) for the root agent and give it permission to
   create and write repos in your org; make a token and wire it on the box.
6. **Install Claude Code as that user, log in once, clone this repo, start a session, and say:**
   *"Read README_AGENT.md and bootstrap the system."*

From there the agent scaffolds the org repos from the templates here, provisions the host, and brings
its own sessions up. You stay in the loop at the marked human-confirm checkpoints (secrets, going live).

> Prefer reading over running a magic script? That's the point — `README_AGENT.md` is a procedure a
> human can follow line by line just as well as an agent can.

## Repository map

```
README.md                 ← you are here (for humans)
README_AGENT.md           ← the bring-up procedure (for the root agent, or a human)
docs/                     ← the write-up: how and why it works
templates/
  infra/                  ← the host layer (root-owned wrapper, systemd, monitor, provisioning)
  kb-agent-shared/        ← shared governance (policies, templates, runbooks) every agent reads
  kb-agent-template/      ← one agent brain scaffold (SOUL.md + OPERATING.md + CLAUDE.md/AGENTS.md adapters)
LICENSE                   ← MIT
```

Full docs index: [`docs/`](docs/README.md). Start with
[`docs/architecture.md`](docs/architecture.md) for the full picture, then
[`docs/config-model.md`](docs/config-model.md) for the portability/security boundary that makes it
tick.

## Status & scope

The **happy path is Claude Code on a Debian-based VPS**. The templates are the same shape running in production for
the reference deployment. Adapt the placeholders (`<ORG>`, `<VPS_HOST>`, `<AGENT>`, …) to your setup.
The application layer (Dokploy / n8n / Cloudflare) is **documented, not templated** here — see
[`docs/app-layer.md`](docs/app-layer.md).

## License

[MIT](LICENSE). Use it, fork it, make it your own. Attribution appreciated, not required.
