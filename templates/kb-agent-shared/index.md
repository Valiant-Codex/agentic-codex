---
type: bundle-index
title: Shared Governance Layer Index
description: Top-level navigation map for your org's shared governance layer.
okf_version: "0.1"
tags:
- index
- shared
- okf
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Shared Governance Layer Index

Start with [`bootstrap.md`](bootstrap.md). Use this file only when you need to navigate the shared layer.

## Start Here

- [`bootstrap.md`](bootstrap.md) — minimal shared startup context and current ecosystem state.
- [`README.md`](README.md) — purpose, operating model, and repository shape.

## Areas

- [`policies/`](policies/) — global policies that apply to every agent.
- [`runbooks/`](runbooks/) — operational runbooks (e.g. agent operations, portability, and VPS recovery).
- [`decisions/`](decisions/) — cross-agent / ecosystem decision records (starts empty; add your own).
- [`handoffs/`](handoffs/) — cross-agent handoff channel (specs/deliverables passed between agents).
- [`templates/`](templates/) — reusable OKF templates for new documents.

## Note

This repository is reached through a `shared` symlink to a sibling clone (`../kb-agent-shared`) in each
agent's own repository (`kb-agent-<role>-<name>`), not as a git submodule. Agent-specific identity,
memory, skills, and tools live in that agent's repo, not here.
