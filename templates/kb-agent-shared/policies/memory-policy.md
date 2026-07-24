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
timestamp: 2026-07-24T00:00:00Z
---
# Memory Policy

## Purpose

Define how the agent manages durable agent memory across GitHub, the runtime (Claude Code), the business KB, object storage, and future memory/retrieval layers.

## Core Principle

The agent's own `kb-agent-<role>-<name>` repository is its canonical durable brain; global policies, decisions, and templates live in `kb-agent-shared` (this repo).

Runtime memories and external memory systems are useful working layers, but durable agent knowledge should be distilled into GitHub Markdown/OKF files.

Business knowledge belongs in your business KB (e.g. Notion or a kb-business repo), not in any agent repository.

## Memory Layers

| Layer | System | Canonical? | Role |
|---|---|---:|---|
| Conversation context | Current chat/session | No | Immediate interaction state. |
| Runtime memory | Claude Code and future memory tools | No | Continuity, convenience, and recall cache. |
| Source material | Object storage, uploads, raw docs, transcripts | No | Raw material to analyze and distill. |
| Business KB | your business KB (e.g. Notion or a kb-business repo) | Domain-specific | Business knowledge and operational context. |
| Canonical agent brain | the agent's own `kb-agent-<role>-<name>` repo | Yes | Agent identity, durable memory, skills, tools. |
| Shared governance | `kb-agent-shared` | Yes | Global policies, cross-agent decisions, templates, conventions. |
| Retrieval indexes | Vector store / graph store / external memory engine | No by default | Derived search and interface layers unless explicitly promoted by decision. |

## What To Save In Your Agent Repository

Save durable, high-signal knowledge when:

- the owner approves a decision affecting architecture, policy, strategy, or operating model.
- A stable preference or constraint should survive runtime migration.
- A skill or runbook should be reused.
- A tool registry entry governs how the agent should use an integration.
- A historical milestone is important enough to explain future state.
- Existing canonical files are stale or contradicted by current decisions.

Do not save:

- raw transcripts;
- low-signal chat fragments;
- duplicate memory;
- business knowledge or operational context that belongs in your business KB;
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
5. Ask the owner if the change is ambiguous, sensitive, external, risky, or broad.
6. Update the target file and the folder `README.md` in the same change when the file list/status changes.
7. Report what changed and what was skipped.

## Archive And Superseded Rules

Use `status: superseded` when a document remains useful historical context but should no longer govern current behavior.

Use `status: archived` when a document should not be loaded operationally except for historical investigation.

The agent must not load `archive/` in normal operation.

## Runtime Memory Rule

Claude Code and other runtime memories may help continuity, but they do not override the canonical repositories. When runtime memory conflicts with a canonical repository, prefer the canonical repository and create a correction if needed.

## Future Memory Tools

Vector stores, graph stores, or similar external memory systems may be evaluated as interface/retrieval layers.

They do not replace the canonical repositories unless the owner approves a new decision record.
