---
type: memory-policy
title: Memory Policy
description: Defines canonical durable memory, runtime memory, promotion rules, and archive/superseded behavior for the agent.
tags:
- memory
- governance
- github
- canonical-brain
status: active
timestamp: 2026-07-10T00:00:00Z
---
# Memory Policy

## Purpose

Define how the agent manages durable agent memory across GitHub, ChatGPT, Claude Code, Notion, Drive, and future memory/retrieval layers.

## Core Principle

The agent's own `kb-agent-<role>-<name>` repository is its canonical durable brain; global policies, decisions, and templates live in `kb-agent-shared` (this repo).

Runtime memories and external memory systems are useful working layers, but durable agent knowledge should be distilled into GitHub Markdown/OKF files.

Business knowledge belongs in your explicit business source of truth (e.g. a wiki or Notion), not in any agent repository.

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

- `distilled-memory.md` — compact durable memory summary.
- `episodic/*.md` — important historical events and milestones.
- `decisions/*.md` — durable architecture/governance decisions.
- `skills/*.md` — reusable procedures.
- `tools/*.md` — tool registry and permissions.
- `policies/*.md` — global rules.

## Promotion Workflow

1. Identify a candidate memory, decision, skill, runbook, or tool note.
2. Check current files and folder `README.md` maps before creating duplicates.
3. Classify the target: distilled memory, episodic memory, decision, policy, skill, tool, template, or archive.
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

- **Working tier — runtime auto-memory** (Claude Code's per-runtime memory store, git-ignored): fast,
  auto-captured during sessions, a convenient recall cache. It is **not canonical** and does **not**
  travel with the brain across a re-provision or a framework switch.
- **Canonical tier — the git brain** (`memory/distilled-memory.md` + `memory/episodic/`): reviewed,
  portable, and inspectable/editable by <OWNER> in GitHub.

High-signal facts must be **distilled from the working tier into the canonical tier** via the Promotion
Workflow above, so the durable brain — not a runtime cache — is the source of truth (<OWNER>'s priorities:
everything editable in GitHub; plug-and-play portability). Run this distillation periodically; the
`agent-audit` skill drives the cadence with <OWNER> in the loop. When a runtime cache conflicts with git,
git wins.

## Future Memory Tools

Basic Memory, Mem0/OpenMemory, Cognee, Letta, vector stores, graph stores, or similar systems may be evaluated as interface/retrieval layers.

They do not replace the canonical repositories unless <OWNER> approves a new decision record.
