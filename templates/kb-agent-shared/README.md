---
type: repository-index
title: Repository Index
description: Root README for the shared governance layer used by all your org's agents.
tags:
- okf
- repository
- shared
status: active
timestamp: 2026-07-24T00:00:00Z
---
# kb-agent-shared

Lean, versioned **shared governance layer** for all of `<ORG>`'s agents.

## Purpose

This repository owns the durable, reviewable, portable knowledge that must be **identical across every
agent**: global policies, ecosystem state, conventions, cross-agent decision records, and OKF templates.

It is **not** any single agent's brain. Each agent's identity, memory, skills, and tool notes live in
that agent's own repository (`kb-agent-<role>-<name>`), which reaches this repo through a `shared`
symlink to a **sibling clone** at `../kb-agent-shared` (not a git submodule — see
the Agentic Codex docs (`config-model.md`)). New agents are scaffolded from a `kb-agent-template`.

Business knowledge and operational context belong in your business KB (e.g. Notion) or another explicit
business source of truth, not in any agent repository.

## Operating Model

- `kb-agent-shared` owns what must stay consistent across agents; each agent repo owns its own brain.
- Runtime memory (LLM runtime / vector stores) is cache/working memory, not the final source of truth.
- Global policies live in `policies/`.
- Cross-agent / ecosystem decisions live in `decisions/`; agent-specific decisions live in that agent's repo.
- Folder-level `README.md` files are required navigation maps and must be updated whenever files in that
  folder are added, removed, renamed, archived, or superseded.

## Repository Shape

- `bootstrap.md` — minimal shared startup contract + ecosystem state.
- `index.md` — top-level navigation map.
- `policies/` — global policies.
- `decisions/` — cross-agent / ecosystem decision records, including superseded history.
- `templates/` — reusable OKF document templates.
- `archive/` — historical material not loaded operationally.

## Agent Repositories

Give each agent a short, memorable name, and pick a small set of archetypes. This template ships with
three:

| Repo | Archetype | Role |
|---|---|---|
| `kb-agent-cos-<name>` | **cos-agent** | Broad, unprivileged Chief of Staff / orchestration |
| `kb-agent-ops-<name>` | **root-agent** | Narrow, privileged infrastructure/operations on `<VPS_HOST>` (sudo) |
| `kb-agent-dev-<name>` | **dev-agent** | Build/deploy engineering; self-contained token + deploy rights, never root |
| `kb-agent-template` | — | Scaffold for new agents |

## Agent Naming Convention

- Agent identity: `role_name` — e.g. `chief-of-staff_<name>`.
- Agent repository: `kb-agent-<role-abbrev>-<name>` — e.g. `kb-agent-cos-<name>`, `kb-agent-ops-<name>`.

## Standard Agent Repository Shape

```text
kb-agent-<role>-<name>/
├── CLAUDE.md            # bootstrap pointer for the runtime
├── SOUL.md + OPERATING.md
├── context/            # (optional)
├── memory/
├── tools/
├── skills/
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

Every Markdown document uses OKF-inspired frontmatter. **Required:** `type`, `title`, `status`
(`active` / `draft` / `superseded` / `archived`), `timestamp` (ISO 8601). **Recommended:** `description`,
`tags`, `supersedes` / `superseded_by`, `resource`. Do not use `updated`; use `timestamp`.

## Safety Boundary

Do not commit secrets, API keys, tokens, passwords, private keys, credentials, raw client data, or
complete raw logs to any repository.
