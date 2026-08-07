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
timestamp: 2026-08-07T00:00:00Z
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
- `type` — concept kind, lowercase, **from the closed vocabulary below**. Put any finer distinction in
  `tags`, never in `type`.

### The `type` vocabulary (closed — do not invent values)

| `type` | Use for |
|---|---|
| `skill` | anything at `skills/<name>/SKILL.md`. Always this, no exceptions. |
| `decision` | a decision record in `decisions/`. |
| `policy` | a fleet rule in `policies/`. |
| `memory` | any memory document, including `distilled-memory.md`. |
| `tool-registry` | a tool/MCP note in `tools/`. |
| `template` | a blank skeleton in `templates/`. |
| `reference` | stable background material (`owner-profile.md`, context docs). |
| `directory-readme` | the README that indexes a directory. Every one of them. |
| `working-doc` | scratch reasoning in `staging/`, until it closes and becomes a decision. |

Consolidated 2026-08-07 from **21** observed values. `runbook` was retired in the same pass: the
directory it named is gone, because only skills are discovered by the runtime — a procedure filed
anywhere else is reachable only by luck. The fleet had reached seven different types for the
one artefact "README of a folder" (`directory-readme`, `directory-index`, `repository-index`,
`bundle-index`, `memory-registry`, `skill-registry`, `tool-registry`), a 16/12 split between `decision`
and `decision-record` with no rule distinguishing them, and `distilled-memory` in three repos against
`memory` in a fourth. A vocabulary that grows one value per document is not a vocabulary.

### Two deliberate exemptions

These are **not** OKF documents and must not carry frontmatter:

1. **`CLAUDE.md`** — a runtime artefact, injected verbatim into the model's context. YAML at the top
   would be noise inside the always-on budget, not metadata for a reader.
2. **`memory/auto/**`** — machine-owned. `memory-mirror` copies these byte-for-byte from the runtime's
   own store, so they carry *its* schema (`name`, `description`, `metadata.node_type`). Adding OKF fields
   by hand is not merely pointless, it is **undone by the next nightly run**. Byte-identity is the
   feature: it is what makes restoring the tier a copy rather than a migration. Do not "fix" these files,
   and do not count them as violations.

Before 2026-08-07 these two exemptions were unwritten, so 31% of the fleet's Markdown "violated" a rule
that was never intended to reach it. Excluding them, conformance is near-total.

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
Include the verified commit or artifact path in the final report when practical. This specializes
`approval-policy.md`'s general commit-and-push rule for the KB-document case — that policy is the
canonical statement of "commit and push are one atomic unit," not repeated here.

## Related

- `shared/policies/approval-policy.md` — §Preferred Git Workflow: the general commit-and-push rule
  (applies to any repo change, not just KB documents) that this section specializes.
- `shared/policies/memory-policy.md` — the Promotion Workflow (deciding *whether* and *where*
  something becomes durable knowledge); this skill only governs the document's shape once that
  decision is made.
- `shared/policies/skills-policy.md` — why `type: skill` is fixed for anything under `skills/`.
- `agent-audit` — the human-gated periodic sweep that curates memory/skills using these same
  hygiene rules.
