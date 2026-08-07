---
type: directory-readme
title: Shared Governance Layer Index
description: Top-level navigation map for the <ORG> shared governance layer.
okf_version: "0.1"
tags:
- index
- shared
- okf
status: active
timestamp: 2026-07-20T00:00:00Z
---
# Shared Governance Layer Index

Start with [`bootstrap.md`](bootstrap.md). Use this file only when you need to navigate the shared layer.

## Start Here

- [`bootstrap.md`](bootstrap.md) — minimal shared startup context and current ecosystem state.
- [`README.md`](README.md) — purpose, operating model, and repository shape.

## Areas

- [`policies/`](policies/) — global policies that apply to every agent.
- [`reference/`](reference/) — shared reference notes (e.g. the Claude Code VPS runtime (systemd topic sessions)).
- [`decisions/`](decisions/) — cross-agent / ecosystem decision records, including superseded history.
- *(No `handoffs/` — since 2026-07-25 cross-agent handoffs are relayed by <OWNER> in chat, not stored here.)*
- [`templates/`](templates/) — reusable OKF templates for new documents.
- [`archive/`](archive/) — historical material that should not be loaded by default.

## Note

This repository is reached through a committed `shared/` symlink (to a sibling clone) in each agent's own repository
(`kb-agent-<role>-<name>`). Agent-specific identity, memory, skills, and tools live in that agent's
repo, not here.
