---
type: directory-readme
title: Template Directory README
description: Explains the lean OKF template standard for new knowledge documents.
tags:
- template
- okf
status: active
timestamp: 2026-07-24T00:00:00Z
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
| `index.md` | template-index | active | Historical/secondary template index. |
| `README.md` | directory-readme | active | Local map for templates. |

## OKF Frontmatter Standard

Every Markdown document should include:

**Required:**
- `type` — concept kind, lowercase.
- `title` — human-readable display name.
- `status` — `active`, `draft`, `superseded`, or `archived`.
- `timestamp` — ISO 8601 datetime of last meaningful change.

**Recommended:**
- `description` — one-line summary.
- `tags` — YAML list of short strings.
- `resource` — canonical URI for the underlying asset when useful.
- `supersedes` / `superseded_by` for transitions.

Do not use `updated`; use `timestamp`.

## Maintenance Rule

Update this README whenever template files are added, removed, renamed, archived, or superseded.
