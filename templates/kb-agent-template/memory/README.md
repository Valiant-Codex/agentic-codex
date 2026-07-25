---
type: memory-registry
title: <AGENT> Memory
description: Durable memory for <AGENT> — the canonical (git) tier of a two-tier model.
tags: [<AGENT>, memory]
status: active
timestamp: 2026-01-01T00:00:00Z
---
# <AGENT> Memory

Durable, high-signal memory for <AGENT>. This is the **canonical (git) tier**: the runtime's own
auto-memory is a fast working cache that does not survive a re-provision or a framework switch, so
high-signal facts are distilled from it into here. See `shared/policies/memory-policy.md` and
`docs/memory.md`. **Never** store secrets, credentials or raw logs here.

| File | Purpose |
|---|---|
| `distilled-memory.md` | Compact durable memory — the file loaded every session. Keep it lean and high-signal. |
| `episodic/` | Time-bounded incident/milestone notes (`YYYY-MM-DD-slug.md`), added as real history accumulates. |
