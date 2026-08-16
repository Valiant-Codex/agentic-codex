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

The **whole always-on contract lives in `CLAUDE.md`** — identity and voice, scope, threat model, gates —
in framework-agnostic Markdown. It is symlinked to `~/CLAUDE.md` and written with absolute
`~/github/...` paths so it resolves from any cwd, and it is **the only file the runtime loads by
itself**. That is why there is no second identity file: a layer that loads only when the model chooses
to obey an instruction is a suggestion, not a layer. The brain *content* (contract, memory, skills,
tools) is plain Markdown and portable to another runtime; the wiring is Claude-Code-specific. See
docs/portability.md in the agentic-codex repo.

## Repo shape

| Path | Purpose |
|---|---|
| `CLAUDE.md` | **The whole always-on contract** — identity, voice, scope, gates, threat model. Symlinked to `~/CLAUDE.md`; the only file the runtime loads. |
| `deploy/topics.tsv` | Remote-Control topic sessions (`key<TAB>Display Name`). |
| `deploy/claude-settings.json` | Curated, portable runtime settings (permissions, notifications). |
| `.mcp.json` | MCP server structure with `${ENV}` placeholders — **no secrets**. Documentation of intent: it is only *read* if the session's cwd is this repo, or if it is passed explicitly with `--mcp-config`. See docs/context-budget.md. |
| `tools/` | This agent's tool notes. |
| `skills/` | This agent's skills, **folder-per-skill** (`<name>/SKILL.md`); fleet-common ones symlink `shared/skills/*`. See docs/skills.md in the agentic-codex repo. |
| `memory/` | This agent's durable memory — two tiers: `distilled-memory.md` (hand-curated standing decisions) and machine-mirrored `auto/`. See docs/memory.md in the agentic-codex repo. |
| `shared/` | Symlink → sibling clone `../kb-agent-shared` (global policies, decisions, runtime reference). |

## The `shared` symlink (sibling clone, not a submodule)

The shared governance layer is a **standalone sibling clone**, not a git submodule. Each brain reaches
it through a committed symlink:

```
shared -> ../kb-agent-shared
```

Clone both repos as siblings under `~/github/<ORG>/`, and a sync timer fast-forwards them (no submodule
commands). If `shared/` does not resolve, clone `<ORG>/kb-agent-shared` next to this repo. See
docs/config-model.md in the agentic-codex repo for why sibling-clone
over submodule.

## How a root agent scaffolds a new brain

1. Copy this template to a new repo `<ORG>/kb-agent-<ROLE>-<AGENT>`.
2. Replace every placeholder in the table above across all files.
3. Confirm the committed `shared` symlink resolves to the sibling `../kb-agent-shared` clone.
4. `provision-agent` symlinks the root `CLAUDE.md` to `~/CLAUDE.md` and installs
   `deploy/claude-settings.json` into the runtime's settings location.
5. Provide real secrets **out of git** (an untracked env file the runtime reads for `${ENV}` values in
   `.mcp.json`).

## Secrets

Never commit secrets. `.gitignore` excludes env files and volatile runtime state; `.mcp.json` carries
only `${ENV}` placeholders resolved at runtime. Real credentials live on the box, out of git.
