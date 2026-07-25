---
type: directory-readme
title: kb-agent-template
description: Template for a single agent's durable brain repository, scaffolded by a root agent into <ORG>/kb-agent-<ROLE>-<AGENT>.
tags:
- template
- agent
- brain
status: active
timestamp: 2026-01-01T00:00:00Z
---
# kb-agent-template — a single agent's brain

This is the **template for one AI agent's durable brain**. A root agent copies it to create each new
agent, producing a repo named `kb-agent-<ROLE>-<AGENT>` (e.g. `kb-agent-ops-<AGENT>`) under your GitHub
org, cloned on the box at `~/github/<ORG>/kb-agent-<ROLE>-<AGENT>/`.

The template is written for a **root-agent archetype** (a privileged infra/ops operator with sudo) as
the worked example, entirely with placeholders. Fill them in and the same shape works for any role.

## Placeholders to replace

| Placeholder | Meaning |
|---|---|
| `<ORG>` | Your GitHub org that owns the repos. |
| `<OWNER>` | The human owner the agent takes instructions from. |
| `<AGENT>` | This agent's short name (and its Unix user). |
| `<ROLE>` | Role slug — `ops`, `cos`, `dev`, … |
| `<VPS_HOST>` | Hostname of the box it runs on. |
| `<TZ>` | Timezone (e.g. `Europe/Rome`). |

## The portability pattern

The **canonical identity + operating contract lives in `SOUL.md`** (who the agent *is* — durable
identity, voice, principles) **and `OPERATING.md`** (what it *does* — scope, threat model, gates,
bootstrap) — framework-agnostic Markdown. The two adapters are thin pointers back to them, so the same
brain runs under different agent runtimes without forking its identity:

- `CLAUDE.md` — adapter for **Claude Code**.
- `AGENTS.md` — adapter for other frameworks that read `AGENTS.md` (Codex, Cursor, …).

Both are short and both say: *identity is in `SOUL.md`, operating contract in `OPERATING.md`; read them
first.* Edit identity/contract in those files; never let the adapters drift into a second source of
truth. Splitting durable *who-you-are* from mutable *what-you-do* is deliberate: it keeps identity from
drifting when operational instructions change. See the Agentic Codex write-up (`docs/portability.md`).

## Repo shape

| Path | Purpose |
|---|---|
| `SOUL.md` | **Canonical** durable identity: who the agent is, voice, principles (loaded every session). |
| `OPERATING.md` | **Canonical** operating contract: mission/scope, threat model, human-confirm gates, bootstrap. |
| `CLAUDE.md` | Thin Claude Code adapter → `SOUL.md` + `OPERATING.md` + `shared/owner-profile.md`. |
| `AGENTS.md` | Thin generic adapter → `SOUL.md` + `OPERATING.md` + `shared/owner-profile.md`. |
| `deploy/home-CLAUDE.md` | The `~/CLAUDE.md` bootstrap symlinked into the home dir. |
| `deploy/topics.tsv` | Remote-Control topic sessions (`key<TAB>Display Name`). |
| `deploy/claude-settings.json` | Curated, portable runtime settings (permissions, notifications). |
| `.mcp.json` | MCP server structure with `${ENV}` placeholders — **no secrets**. |
| `tools/` | This agent's tool notes. |
| `skills/` | This agent's skills, **folder-per-skill** (`<name>/SKILL.md`); fleet-common ones symlink `shared/skills/*`. See `docs/skills.md`. |
| `memory/` | This agent's durable memory — two-tier (`distilled-memory.md` + `episodic/` + machine-mirrored `auto/`). See `docs/memory.md`. |
| `context/` | Working context and scratch notes. |
| `shared/` | Symlink → sibling clone `../kb-agent-shared` (global policies, decisions, runtime reference). |

## The `shared` symlink (sibling clone, not a submodule)

The shared governance layer is a **standalone sibling clone**, not a git submodule. Each brain reaches
it through a committed symlink:

```
shared -> ../kb-agent-shared
```

Clone both repos as siblings under `~/github/<ORG>/`, and a sync timer fast-forwards them (no submodule
commands). If `shared/` does not resolve, clone `<ORG>/kb-agent-shared` next to this repo. See
the Agentic Codex write-up (`docs/config-model.md` in the agentic-codex repo) for why sibling-clone
over submodule.

## How a root agent scaffolds a new brain

1. Copy this template to a new repo `<ORG>/kb-agent-<ROLE>-<AGENT>`.
2. Replace every placeholder in the table above across all files.
3. Create the sibling `shared` symlink and confirm it resolves to `../kb-agent-shared`.
4. Install `deploy/home-CLAUDE.md` as `~/CLAUDE.md`, and `deploy/claude-settings.json` into the
   runtime's settings location.
5. Provide real secrets **out of git** (an untracked env file the runtime reads for `${ENV}` values in
   `.mcp.json`).

## Secrets

Never commit secrets. `.gitignore` excludes env files and volatile runtime state; `.mcp.json` carries
only `${ENV}` placeholders resolved at runtime. Real credentials live on the box, out of git.
