---
type: directory-readme
title: Repository Index
description: Root README for the shared governance layer used by all <ORG> agents.
tags:
- okf
- repository
- shared
status: active
timestamp: 2026-08-11T00:00:00Z
---
# <ORG> — kb-agent-shared

Lean, versioned **shared governance layer** for all <ORG> agents.

## Purpose

This repository owns the durable, reviewable, portable knowledge that must be **identical across every
agent**: global policies, ecosystem state, conventions, cross-agent decision records, and OKF templates.

It is **not** any single agent's brain. Each agent's identity, memory, skills, and tool notes live in
that agent's own repository (`kb-agent-<role>-<name>`), which reaches this repo through a `shared`
symlink to a **sibling clone** at `../kb-agent-shared` (not a git submodule — see
`decisions/2026-07-21-shared-as-sibling-clone-not-submodule.md`). New agents are scaffolded from
`kb-agent-template`.

Business knowledge, client work, offers, CRM notes, delivery processes, and operating dashboards belong
in Notion or another explicit business source of truth, not in any agent repository.

## Operating Model

- `kb-agent-shared` owns what must stay consistent across agents; each agent repo owns its own brain.
- Runtime memory (ChatGPT / Claude Code / vector stores) is cache/working memory, not the final source of truth.
- Global policies live in `policies/`.
- Cross-agent / ecosystem decisions live in `decisions/`; agent-specific decisions live in that agent's repo.
- Folder-level `README.md` files are required navigation maps and must be updated whenever files in that
  folder are added, removed, renamed, archived, or superseded.

## Repository Shape

- `bootstrap.md` — minimal shared startup contract + ecosystem state.
- `index.md` — top-level navigation map.
- `policies/` — global policies.
- `skills/` — fleet-common skills, symlinked into every agent's `skills/`. **The
  highest-consequence directory here:** a `SKILL.md` is instructions that reach every agent's
  context — including your privileged agent's — through kb-sync, unreviewed.
- `decisions/` — cross-agent / ecosystem decision records, including superseded history.
- `templates/` — reusable OKF document templates.
- `archive/` — historical material not loaded operationally.

## Agent Repositories

| Repo | Agent | Role |
|---|---|---|
| `kb-agent-<role>-<cos-agent>` | `chief-of-staff_<name>` | Broad, unprivileged Chief of Staff / orchestration |
| `kb-agent-<role>-<root-agent>` | `infra-ops_<name>` | Narrow, privileged infrastructure/operations on the host |
| `kb-agent-template` | — | Scaffold for new agents |

## Agent Naming Convention

- Agent identity: `role_name` — e.g. `chief-of-staff_<name>`.
- Agent repository: `kb-agent-<role-abbrev>-<name>` — e.g. `kb-agent-<role>-<cos-agent>`, `kb-agent-<role>-<root-agent>`.

## Standard Agent Repository Shape

```text
kb-agent-<role>-<name>/
├── CLAUDE.md           # the whole always-on contract; symlinked to ~/CLAUDE.md
├── memory/             # distilled-memory.md (human-curated) + auto/ (machine-mirrored)
├── tools/
├── skills/             # <name>/SKILL.md per skill; shared ones are symlinks into shared/skills/
├── deploy/             # Tier-3 payload applied by provision-agent (topics.tsv, settings)
├── .mcp.json           # MCP structure only — secrets via ${ENV}, never committed
└── shared/             # symlink → ../kb-agent-shared (sibling clone)
```

## Less Is More

Keep the knowledge base small, clear, and easy to rewrite.

- Prefer updating an existing document over creating a new one.
- Consolidate overlapping notes quickly, and keep each fact in exactly one owning place.
- Mark stale decisions `superseded` instead of pretending they are current.
- Move material that should not be loaded operationally to `archive/` or mark it `archived`.

## OKF Frontmatter

Defined once, in `skills/knowledge-governance-workflow/SKILL.md`, and asserted by
`agentic-divergence-check`. Not restated here — this section used to hold a third copy of the
contract (after the skill and `templates/README.md`), all three disagreeing on which fields are
required and only one of them checked by anything.

## Safety Boundary

Do not commit secrets, API keys, tokens, passwords, private keys, credentials, raw client data, or
complete raw logs to any repository.
