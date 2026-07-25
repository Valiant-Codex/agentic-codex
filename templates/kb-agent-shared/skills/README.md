---
type: skill-registry
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

Skills that every agent uses. Each lives here once (canonical) and is symlinked into each agent's repo
(`ln -s ../shared/skills/<name> skills/<name>`), so it is harness-discoverable via each agent's whole-dir
`~/.claude/skills` symlink without duplication. Propagated to all clones by kb-sync.

| Skill | Purpose |
|---|---|
| `skillify/` | Author, update, or retire an agent skill in the v2 format (folder + SKILL.md, progressive disclosure, lifecycle). The executable companion to `policies/skills-policy.md`. |
| `agent-audit/` | Interactive, human-in-the-loop tune-up of one agent's brain with <OWNER> (skills, SOUL, OPERATING, memory). The only path that changes capability/identity; consumes the nightly `dream-log.md` suggestions. |

See `policies/skills-policy.md` for the authoring rules and lifecycle.

## Maintenance Rule

Update this README whenever fleet-common skills are added, removed, renamed, archived, or superseded.
