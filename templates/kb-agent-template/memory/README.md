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
| `distilled-memory.md` | **Standing decisions and closed questions** — what `auto/` structurally cannot hold: the things a fresh session would otherwise re-litigate. Hand-curated. Not loaded automatically; nothing but `CLAUDE.md` is. |
| `auto/` | **Machine-owned.** A nightly `memory-mirror` rsyncs the runtime's own auto-memory here and commits it, so the fast tier survives a re-provision. **Do not hand-edit** — your edits are overwritten by the next mirror. Edit the runtime's memory instead. |

The two tiers answer different questions: `auto/` is *what the runtime happened to record*, complete
but unranked; `distilled-memory.md` is *what you decided is worth carrying*, ranked but only as fresh
as its last curation. Mirroring is automatic, distilling is not — so schedule the distil pass, or the
curated tier quietly falls months behind the one nobody reads.
