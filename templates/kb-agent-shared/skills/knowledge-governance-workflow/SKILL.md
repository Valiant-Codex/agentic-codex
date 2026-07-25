---
name: knowledge-governance-workflow
description: OKF frontmatter and hygiene rules for any agent's own KB — required/recommended frontmatter fields, a pre-commit validation checklist, and post-write verification for external repo writes. Use whenever adding, editing, or archiving a Markdown document in an agent brain repo.
type: skill
title: Knowledge Governance Workflow — OKF Hygiene
tags:
- skill
- governance
- okf
- lean
- fleet
status: active
timestamp: 2026-07-25T00:00:00Z
---
# Knowledge Governance Workflow — OKF Hygiene

## Purpose

Keep every agent brain small, current, and OKF-conformant. This is the *document-format* half of
KB governance — the frontmatter shape and the checks that catch drift before it's committed. The
*decision-making* half (identify a candidate, classify its target, decide whether it's durable,
promote or discard) is `shared/policies/memory-policy.md`'s Promotion Workflow — don't duplicate
it here; read that policy for "where does this fact go."

## OKF Adherence

Every Markdown document in an agent repo adheres to the [Open Knowledge Format v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).

## OKF Frontmatter Standard

Every Markdown document must include a parseable YAML frontmatter block with at minimum:

**Required (OKF spec §4.1):**
- `type` — concept kind (lowercase, e.g. `policy`, `decision`, `template`; for anything under
  `skills/<name>/SKILL.md` this is always `skill` — see `skills-policy.md` — put any finer
  distinction in `tags`, not `type`).

**Recommended (OKF spec §4.1):**
- `title` — human-readable display name.
- `description` — one-line summary.
- `tags` — YAML list of short strings.
- `timestamp` — ISO 8601 datetime of last meaningful change.
- `resource` — canonical URI, when the document describes a specific system/tool/resource (omit
  for abstract concepts).

**Producer extensions (permitted by OKF spec §4.1):**
- `status` — `active`, `draft`, `superseded`, or `archived`.

## Core Rules

- Prefer one clear document over several overlapping ones.
- Update an existing file before creating a sibling that says almost the same thing.
- Rewrite or archive stale content quickly rather than letting it sit contradicted.
- Keep only durable knowledge that improves future execution.
- Avoid unnecessary metadata churn (don't bump `timestamp` for non-meaningful edits).

## Validation

Before committing a KB change, verify that:

- frontmatter is valid YAML;
- `type` is present, non-empty, and correct for the file's location (`skill` under any `skills/`
  folder, per `skills-policy.md`);
- `timestamp` is ISO 8601;
- local links resolve;
- no secrets or raw sensitive data are included;
- the diff reduces drift rather than adding noise (fewer/clearer documents, not more).

## Post-Write Verification

For writes that land in a remote repo, don't report completion until the remote source of truth
has been verified after the write. Acceptable verification: checking the pushed commit on the
remote branch, reading the changed file through GitHub, or the GitHub Contents API or equivalent.
Include the verified commit or artifact path in the final report when practical.

## Related

- `shared/policies/memory-policy.md` — the Promotion Workflow (deciding *whether* and *where*
  something becomes durable knowledge); this skill only governs the document's shape once that
  decision is made.
- `shared/policies/skills-policy.md` — why `type: skill` is fixed for anything under `skills/`.
- `agent-audit` — the human-gated periodic sweep that curates memory/skills using these same
  hygiene rules.
