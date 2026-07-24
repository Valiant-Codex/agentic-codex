---
type: policy
title: Source Of Truth Policy
description: Defines canonical systems for durable agent memory, business knowledge, runtime context, and approvals.
tags:
- policy
- source-of-truth
- governance
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Source Of Truth Policy

## Purpose

Define which system is authoritative for each class of information used by the agent.

## Canonical Systems

| Class | Canonical System | Notes |
|---|---|---|
| Agent identity, prompts, context, durable memory, skills, tool registry | the agent's own `kb-agent-<role>-<name>` repo | Canonical durable brain for that agent. |
| Global policies, cross-agent decisions, templates, conventions | `kb-agent-shared` | Shared governance layer, included as a `shared/` clone/symlink in each agent repo. |
| Business knowledge, approved business processes, offer design, client-facing methods | your business KB (e.g. Notion or a kb-business repo) | Business KB is separate from agent brain. |
| Business workspace, dashboard, CRM/light delivery operations | your business KB (e.g. Notion or a kb-business repo) | Operational business interface, not canonical agent memory. |
| Task and work management | your work tracker (e.g. Jira or Linear) | Keep work tracking out of the agent brain; use a dedicated tracker that is resellable/consultable for clients. |
| Raw files, artifacts, uploaded docs, research source material | object storage or direct uploads | Source material only; promote distilled knowledge into canonical locations. |
| Runtime/interface | Claude Code (on the VPS) | Interaction, reasoning, tool execution, channel layer. Not durable canonical memory. |
| Runtime memories, vector stores, graph stores, external memory engines | Derived/cache/interface layers | Useful for retrieval and continuity; not source of truth unless explicitly promoted by decision record. |
| Approvals, strategic direction, policy changes | the owner | Final authority. |

## Operating Rule

When systems disagree, prefer the canonical source for that information class and create a correction or decision record if the disagreement matters.

## Archive Rule

Archived and superseded material must not be loaded for normal operation. Read it only when investigating history, migrations, or why a decision changed.

## Citations

[1] [Open Knowledge Format v0.1 specification](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)
