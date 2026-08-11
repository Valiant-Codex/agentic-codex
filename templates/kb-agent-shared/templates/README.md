---
type: directory-readme
title: Template Directory README
description: Explains the lean OKF template standard for new knowledge documents.
tags:
- template
- okf
status: active
timestamp: 2026-08-11T00:00:00Z
---
# Templates

## Purpose

Templates define the default shape for new documents in this bundle.

Use the closest template only when an existing document cannot be expanded cleanly. Prefer updating the owning document over creating unnecessary new files.

## Files

| File | Type | Status | Purpose |
|---|---|---|---|
| `agent-template.md` | template | active | Template for a new agent folder/prompt shape. |
| `concept-template.md` | template | active | Generic concept document template. |
| `decision-template.md` | template | active | Decision record template. |
| `memory-entry-template.md` | template | active | Durable memory entry template. |
| `process-template.md` | template | active | Process/runbook-style template. |
| `skill-template.md` | template | active | Reusable skill template. |
| `tool-registry-template.md` | template | active | Tool registry entry template. |
| `README.md` | directory-readme | active | Local map for templates. |

## OKF Frontmatter Standard

**The contract lives in exactly one place: `skills/knowledge-governance-workflow/SKILL.md`.**
Read it there — required and recommended fields, the closed `type` vocabulary, and the two
directories exempt by design.

It is deliberately not restated here. This section used to carry its own copy, and being the copy
new documents are scaffolded against, it was the copy that propagated: it listed a different
required set and asked for a "concept kind, lowercase" with **no vocabulary at all** — the exact
drift the closed list exists to stop. A second copy of a contract is not redundancy, it is a
slower-moving contradiction, and the one your tooling does not assert always wins in practice.

## Maintenance Rule

Update this README whenever template files are added, removed, renamed, archived, or superseded.
