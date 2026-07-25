---
type: policy
title: Skills Policy
description: How agents author and maintain their own skills — self-writing capability kept controlled by git, scope, a template, and pruning. v2 format (folder + SKILL.md, progressive disclosure).
tags:
- policy
- skills
- governance
- agents
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Skills Policy

## Purpose

Agents may **write and maintain their own skills** — reusable procedures they perform repeatedly. This is a deliberate capability (an agent that codifies its own know-how gets better over time), kept **controlled** so it never drifts into an unreviewable, sprawling mess. The control is structural: git, scope, a template, and pruning. The executable companion to this policy is the `skillify` skill.

## Model — config as code

- A skill's **canonical copy** is a **folder** in the agent's own repo at `<agent-repo>/skills/<name>/`, whose entry point is `SKILL.md` (Anthropic Agent Skills format), versioned in git. Optional `references/` (deep detail, loaded on demand) and `scripts/` (executable helpers; only their output enters context) subdirectories provide progressive disclosure. Keep the `SKILL.md` body under ~5k tokens.
- **Fleet-common skills** — used by all agents (e.g. `skillify`, `agent-audit`) — live once in `kb-agent-shared/skills/<name>/` and are symlinked into each agent's tree (`ln -s ../shared/skills/<name> skills/<name>`): one canonical copy, propagated by kb-sync, no duplication.
- **Registration is automatic.** `~/.claude/skills` is a **whole-directory symlink** to the agent's repo `skills/`, so any `skills/<name>/SKILL.md` — including symlinked fleet-common skills — is harness-discoverable with zero per-skill setup. (This replaces the old per-skill `~/.claude/skills/<name>/SKILL.md` symlink model.)
- **Frontmatter** must include `name` and `description` (harness discovery; `description` states *what* the skill does **and when** to use it). The repo copy also carries OKF fields (`type: skill`, `title`, `tags`, `status`, `timestamp`). Use `templates/skill-template.md` as the blank skeleton.

## Guardrails (why this stays controlled)

1. **Git is the control plane.** Every skill change is a commit — diffable, reviewable, and revertible by <OWNER>. Nothing self-modifies invisibly. This is precisely what unbounded self-writing agents lack.
2. **Write real, not speculative.** One skill = one procedure the agent actually performs. Don't pre-write skills for things that haven't happened.
3. **Scope correctly.** An agent writes agent-specific skills only in its own `kb-agent-<role>-<name>`. Genuinely fleet-common procedures go in `kb-agent-shared/skills/` (symlinked into each agent). Cross-agent or privileged procedures live in the repo of the agent that owns them (e.g. agent lifecycle → the ops agent's `manage-agents`).
4. **Prune and mark, don't accrete.** Review skills periodically; when a procedure changes, update the skill; when it's obsolete, mark it **superseded** in place (a banner at the top pointing at the replacement, so the reasoning survives) or move the folder to `skills-archive/` (outside `skills/`, so the harness stops loading it) rather than deleting silently.
5. **Keep them lean.** A skill points to `shared/runbooks/*` for deep detail instead of duplicating it; it sequences the job and records the gotchas.

## Approval

Authoring and updating skills in an agent's own repo is an internal-KB action allowed without approval (see `approval-policy.md`). Deleting a skill, or any skill with external side effects, follows the approval policy. <OWNER> can veto or roll back any skill via git history at any time.
