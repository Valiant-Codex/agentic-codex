<!-- title: Reference architecture — the deployment this came from -->
# Reference architecture

Agentic Codex is the distilled, genericized version of a system that runs in production for
**[Valiant Codex](https://github.com/Valiant-Codex)**. This page describes that concrete deployment as
a worked example, so the abstract templates have a real referent. (No hostnames, tokens, emails, or
other secrets appear here — those never leave the private side.)

## The fleet

Three agents run as Claude Code sessions on one Debian-based VPS, each a distinct Unix user with its own
GitHub bot account and brain repo. They're named after Tolkien's smiths and delvers — a small,
memorable convention, not a requirement.

| Agent | Archetype | Privilege | What it owns |
|---|---|---|---|
| **Durin** | root-agent | `sudo` — owns the host | Infra/ops: users, packages, Docker & Dokploy, Cloudflare zone (Tunnel + Access), backups, provisioning new agents, and the `agentic-monitor` heartbeat. The only privileged principal. |
| **Galadriel** | cos-agent | none | Chief of Staff: research, business reasoning, orchestration, content, intel briefings. Broad tools, no root. |
| **Celebrimbor** | dev-agent | none (own deploy token) | Build/deploy: the marketing website and n8n automation workflows, using its own scoped deploy rights — never routed through root. |

Named for delvers and smiths on purpose: the privileged one, *Durin*, "works beneath everything, where
the foundations are" — a reminder that the privilege is a liability to handle, not a power to enjoy.

## How the layers are instantiated

- **Brains** — `kb-agent-ops-durin`, `kb-agent-cos-galadriel`, `kb-agent-dev-celebrimbor`: each a Git
  repo of OKF-style Markdown (identity, memory, skills, tools), reaching shared governance through a
  `shared -> ../kb-agent-shared` sibling-clone symlink.
- **Governance** — `kb-agent-shared`: the policies, templates, decisions, runbooks, and cross-agent
  handoff channel every agent reads. New agents are scaffolded from `kb-agent-template`.
- **Host** — `infra`: the `claude-topic` wrapper, systemd units, `kb-sync`, `agentic-monitor`,
  `provision-agent`. Installed by Durin.
- **Config model** — the three tiers exactly as in [`config-model.md`](config-model.md): Git is the
  portable truth, `kb-sync` refreshes inert data every 15 min, and Durin's `provision-agent` is the
  only thing that installs executables/settings — the security boundary.

## The application layer, as used

- **Durin** runs **Dokploy** on the VPS and owns the **Cloudflare** zone: services are exposed via
  **Tunnel** (no public inbound ports) with **Access** in front of admin UIs (Dokploy, a Cockpit
  console). Durin also runs the daily VPS backups.
- **Celebrimbor** deploys the website and builds **n8n** workflows with its own deploy token.
- **Vaultwarden** runs as a Dokploy app and holds every agent's secrets, so a box rebuild restores
  secrets from one place ([`secrets.md`](secrets.md)).
- **Monitoring**: `agentic-monitor` heartbeats a **healthchecks.io** check that fans out to
  **Telegram** — silence is the alarm ([`monitoring.md`](monitoring.md)).

## Governance decisions that shaped it

The private side keeps decision records; the load-bearing ones are baked into these docs:

- **GitHub + OKF-structured Markdown as the canonical agent brain** — durable, portable, reviewable,
  low-lock-in.
- **Claude Code on the VPS as the primary runtime** — chosen specifically for Remote Control's
  multi-device reach; no extra gateway; brains kept framework-agnostic anyway.
- **`shared` as a sibling clone, not a submodule** — always-latest governance, no pinning overhead.
- **Provisioning as the portability + security boundary** — the three-tier model.
- **A single host-level dead-man's-switch** for observability — reuse, not a monitoring stack.

## What this shows

The reference deployment is proof the shape holds up in daily use: three agents with different
privileges and lanes, driven from phone and laptop, recoverable from Git + a restored secret, and loud
when something breaks. Your deployment can be a single agent or a larger fleet — the templates and the
config model are the same either way.
