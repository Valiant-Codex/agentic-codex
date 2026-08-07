---
type: policy
title: Memory Policy
description: Defines canonical durable memory, runtime memory, promotion rules, and archive/superseded behavior for the agent.
tags:
- memory
- governance
- github
- canonical-brain
status: active
timestamp: 2026-07-10T00:00:00Z
supersedes:
- decisions/2026-07-03-openclaw-first-memory-stack.md
- decisions/2026-06-28-memory-stack-design.md
---
# Memory Policy

## Purpose

Define how the agent manages durable agent memory across GitHub, ChatGPT, Claude Code, Notion, Drive, and future memory/retrieval layers.

## Core Principle

The agent's own `kb-agent-<role>-<name>` repository is its canonical durable brain; global policies, decisions, and templates live in `kb-agent-shared` (this repo).

Runtime memories and external memory systems are useful working layers, but durable agent knowledge should be distilled into GitHub Markdown/OKF files.

Business knowledge belongs in Notion or another explicit business source of truth, not in any agent repository.

## Memory Layers

| Layer | System | Canonical? | Role |
|---|---|---:|---|
| Conversation context | Current chat/session | No | Immediate interaction state. |
| Runtime memory | ChatGPT memory, Claude Code, future memory tools | No | Continuity, convenience, and recall cache. |
| Source material | Drive, uploads, raw docs, transcripts | No | Raw material to analyze and distill. |
| Business KB | Notion / explicit business source of truth | Domain-specific | Business knowledge, offers, client delivery, CRM/workspace. |
| Canonical agent brain | the agent's own `kb-agent-<role>-<name>` repo | Yes | Agent identity, durable memory, skills, tools. |
| Shared governance | `kb-agent-shared` | Yes | Global policies, cross-agent decisions, templates, conventions. |
| Retrieval indexes | Vector store / graph store / Basic Memory / Mem0 / Cognee / Letta | No by default | Derived search and interface layers unless explicitly promoted by decision. |

## What To Save In Your Agent Repository

Save durable, high-signal knowledge when:

- <OWNER> approves a decision affecting architecture, policy, strategy, or operating model.
- A stable preference or constraint should survive runtime migration.
- A skill or runbook should be reused.
- A tool registry entry governs how the agent should use an integration.
- A historical milestone is important enough to explain future state.
- Existing canonical files are stale or contradicted by current decisions.

Do not save:

- raw transcripts;
- low-signal chat fragments;
- duplicate memory;
- business knowledge, client delivery context, CRM notes, offers, or process docs that belong in Notion;
- unreviewed external claims;
- secrets, credentials, raw private logs, or sensitive client data.

## Memory File Types

- `distilled-memory.md` — standing decisions and closed questions only (see the tier model below).
- `memory/auto/*.md` — machine-owned nightly mirror of the runtime store. **Never hand-edited.**
- `decisions/*.md` — durable architecture/governance decisions.
- `skills/*.md` — reusable procedures.
- `tools/*.md` — tool registry and permissions.
- `policies/*.md` — global rules.

## Promotion Workflow

1. Identify a candidate memory, decision, skill, runbook, or tool note.
2. Check current files and folder `README.md` maps before creating duplicates.
3. Classify the target: a standing decision (distilled memory), a decision record, a policy, a skill, a tool note, or an archive entry. Anything episodic needs no action — the runtime already recorded it and the nightly mirror already carries it.
4. Draft the smallest coherent update.
5. Ask <OWNER> if the change is ambiguous, sensitive, external, risky, client-related, or broad.
6. Update the target file and the folder `README.md` in the same change when the file list/status changes.
7. Report what changed and what was skipped.

## Archive And Superseded Rules

Use `status: superseded` when a document remains useful historical context but should no longer govern current behavior.

Use `status: archived` when a document should not be loaded operationally except for historical investigation.

The agent must not load `archive/` during normal operation.

## Runtime Memory Rule

ChatGPT/Claude Code/other runtime memories may help continuity, but they do not override the canonical repositories. When runtime memory conflicts with a canonical repository, prefer the canonical repository and create a correction if needed.

## Two-Tier Durable Memory

In practice durable memory has two tiers, and the distinction matters for portability:

- **`memory/auto/` — machine-owned, and the tier that carries the weight.** `memory-mirror` rsyncs the
  runtime's own auto-memory here nightly and commits it, so the fast tier survives a re-provision. It is
  a **byte-identical copy**: no transformation, which is exactly what makes restoring it trivial. Do not
  hand-edit it — the next mirror overwrites your edit. To correct something here, correct the runtime
  memory it is copied from.
- **`distilled-memory.md` — hand-curated, and deliberately narrow.** It holds **standing decisions and
  closed questions**: the things a fresh session would otherwise cheerfully re-litigate. It is *not* a
  summary of `auto/`. If a fact appears in both, delete it from here.

The two answer different questions. `auto/` is *what the runtime happened to record* — complete, unranked,
and refreshed for free. `distilled-memory.md` is *what was settled* — ranked, and only as fresh as its last
curation. **Mirroring is automatic; distilling is not**, which is the asymmetry that bites: left alone, the
curated tier silently falls months behind the one nobody has to maintain. So keep the curated tier small
enough that reviewing it is cheap, and refresh it in `agent-audit` with <OWNER> in the loop. When a runtime
cache conflicts with git, git wins.

**On `episodic/`** (dropped 2026-08-07): a third tier of dated incident notes was specified from
2026-06-27, instantiated by exactly one agent, and abandoned after four weeks — no automation ever wrote
it, and nothing new was added for the last 17 days it existed, while five repos went on referencing it and
one described a directory that had never been created. Durable incidents belong in `auto/` (the runtime
records them by itself) or, when they change how the fleet operates, in `decisions/`. A tier nobody writes
is not a tier; it is a claim.

## Future Memory Tools

Basic Memory, Mem0/OpenMemory, Cognee, Letta, vector stores, graph stores, or similar systems may be evaluated as interface/retrieval layers.

They do not replace the canonical repositories unless <OWNER> approves a new decision record.
