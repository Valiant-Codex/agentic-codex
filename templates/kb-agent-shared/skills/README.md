---
type: directory-readme
title: Shared (Fleet-Common) Skills
description: Skills used by every agent, kept once here and symlinked into each agent's skills/ so there is a single canonical copy.
tags:
- skills
- shared
- fleet
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Shared (Fleet-Common) Skills

Skills that every agent uses. All five earned their place in the reference deployment before being
published here — none is speculative. Each lives here once (canonical) and is symlinked into each agent's repo
(`ln -s ../shared/skills/<name> skills/<name>`), so it is harness-discoverable via each agent's whole-dir
`~/.claude/skills` symlink without duplication. Propagated to all clones by kb-sync.

| Skill | Purpose |
|---|---|
| `skillify/` | Author, update, or retire an agent skill in the v2 format (folder + SKILL.md, progressive disclosure, lifecycle). The executable companion to `policies/skills-policy.md`. |
| `agent-audit/` | Interactive, human-in-the-loop tune-up of one agent's brain with <OWNER> (skills, SOUL, OPERATING, memory). The periodic pass where accumulated experience becomes durable change; enforces that no scheduled job alters skills or identity. |
| `advisor-review/` | Get an independent, adversarial second opinion before acting: how to scope a sub-agent so it challenges rather than agrees, the constraints it must always be given, and the rule that you re-verify its load-bearing claims yourself. |
| `decision-loop/` | Sparring procedure for strategic or tactical calls — strawman, attack, converge, record — with dependency-ordered blocks and early falsification of load-bearing assumptions. |
| `knowledge-governance-workflow/` | OKF frontmatter and hygiene rules for any brain: required fields, a pre-commit checklist, and post-write verification. |

See `policies/skills-policy.md` for the authoring rules and lifecycle.

## Maintenance Rule

Update this README whenever fleet-common skills are added, removed, renamed, archived, or superseded.
