<!-- title: Reference architecture — the deployment this came from -->
# Reference architecture

Agentic Codex is the distilled, genericized version of a system that runs in production for
**[Valiant Codex](https://github.com/Valiant-Codex)**. This page describes that concrete deployment as
a worked example, so the abstract templates have a real referent. (No hostnames, tokens, emails, or
other secrets appear here — those never leave the private side.)

## The fleet

Two active agents run as Claude Code sessions on one Debian-based VPS, each a distinct Unix user
with its own GitHub bot account and brain repo, plus two **dormant**. They're named after
Tolkien's smiths, delvers and keepers — a small, memorable convention, not a requirement.

| Agent | Archetype | Privilege | What it owns |
|---|---|---|---|
| **Durin** | root-agent | `sudo` — owns the host | Infra/ops: users, packages, Docker & Dokploy, Cloudflare zone (Tunnel + Access), backups, provisioning new agents, and the `agentic-monitor` heartbeat. The only privileged principal. **Since 2026-08-22 it also owns the application layer** — website, web apps, n8n — building and deploying through an unprivileged `builder` Unix user, never as itself. |
| **Galadriel** | cos-agent | none | Chief of Staff: research, business reasoning, orchestration, content, intel briefings — **and** finance, fiscal compliance, the legal entity and the unified agenda, folded back in 2026-08-03. Broad tools, no root. |
| **Celebrimbor** | dev-agent | none (own deploy token) | **Dormant since 2026-08-22.** Ran build/deploy for the marketing website and n8n workflows with its own scoped deploy rights. The seam that ended it was Cloudflare, touched by both it and the root-agent: every deploy needing a DNS record or an Access policy became a human relay. |
| **Elrond** | cfo-agent | none | **Dormant since 2026-08-03.** Ran finance & admin as a separate agent for five weeks; the split cost more coordination than the separation bought, so the scope went back to Galadriel. |

**Splitting an agent out is reversible, and retiring one is not deleting it.** Elrond's brain repo is
archived, its Unix user and token are preserved, and it stays in the fleet roster — because the
roster drives `kb-sync` and the monitor, and removing a live-on-disk agent from it would alarm every
day. A dormant agent costs almost nothing; an orphaned home directory nobody sweeps costs a lot. The
one thing worth switching off is its share of the unattended writers (the nightly memory mirror),
which otherwise runs forever against a repo that can no longer accept pushes.

Named for delvers and smiths on purpose: the privileged one, *Durin*, "works beneath everything, where
the foundations are" — a reminder that the privilege is a liability to handle, not a power to enjoy.

## How the layers are instantiated

- **Brains** — `kb-agent-ops-durin`, `kb-agent-cos-galadriel`, `kb-agent-dev-celebrimbor`,
  `kb-agent-cfo-elrond`: each a Git
  repo of OKF-style Markdown (identity, memory, skills, tools), reaching shared governance through a
  `shared -> ../kb-agent-shared` sibling-clone symlink.
- **Governance** — `kb-agent-shared`: the policies, templates, decisions, fleet-common skills, and cross-agent
  governance layer every agent reads. New agents are scaffolded from `kb-agent-template`.
- **Host** — `infra`: the `claude-topic` wrapper, systemd units, `kb-sync`, `agentic-monitor`,
  `provision-agent`. Installed by Durin.
- **Config model** — the three tiers exactly as in [`config-model.md`](config-model.md): Git is the
  portable truth, `kb-sync` refreshes inert data every 15 min, and Durin's `provision-agent` is the
  only thing that installs executables/settings — the security boundary.

## The application layer, as used

- **Durin** runs **Dokploy** on the VPS and owns the **Cloudflare** zone: application services are
  exposed via **Tunnel**, bound to `127.0.0.1`, with **Access** in front of admin UIs. Durin also
  runs the daily VPS backups.

  > **Bind the admin plane too, or firewall it.** "Exposed via Tunnel" is a property of the services
  > you deliberately put behind it — it is not a property of the box. In the reference deployment the
  > application containers bind `127.0.0.1`, but the *control plane* did not: Dokploy's own UI, the
  > Cockpit console and the Docker Swarm manager API all listened on `0.0.0.0`, leaving ingress
  > resting entirely on a provider-level cloud firewall that had never been verified. Assume nothing
  > about the layers you did not configure yourself: enumerate what actually listens
  > (`ss -tlnp`), and remember that **a host firewall is not enough** — Docker publishes ports
  > through its own nftables chains and walks straight past UFW's INPUT rules. The layer that can
  > carry the policy is the provider firewall, or a bind address.
- **Celebrimbor** deploys the website and builds **n8n** workflows with its own deploy token.
- **Vaultwarden** runs as a Dokploy app as the owner's secret store (reachable only over the
  tailnet). Being honest about the current state: the agents' own tokens live in per-user files on
  the box (`~/.config/gh/hosts.yml`, `~/.claude.json`, `~/.config/<agent>/secrets.env`) and are
  restored by hand on a rebuild; the box (Vaultwarden included) is covered by daily provider
  snapshots. Consolidating every agent secret into the store, with an off-box export, is the
  documented open item ([`secrets.md`](secrets.md)) — not yet the achieved state.
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

The reference deployment is proof the shape holds up in daily use: agents with different privileges
and lanes, driven from phone and laptop, recoverable from Git + a restored secret, loud when
something breaks — and able to retire one of its own without deleting it. Your deployment can be a single agent or a larger fleet — the templates and the
config model are the same either way.
