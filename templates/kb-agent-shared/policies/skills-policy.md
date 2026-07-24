---
type: policy
title: Skills Policy
description: How agents author and maintain their own skills — self-writing capability kept controlled by git, scope, and pruning.
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

Agents may **write and maintain their own skills** — reusable procedures they perform repeatedly. This is a deliberate capability (an agent that codifies its own know-how gets better over time), kept **controlled** so it never drifts into an unreviewable, sprawling mess. The control is structural: git, scope, a template, and pruning.

## Model — config as code

- A skill's **canonical copy** lives in the agent's own repo at `<agent-repo>/skills/<name>.md`, versioned in git.
- The **executable copy** Claude Code invokes lives at `~/.claude/skills/<name>/SKILL.md`. Prefer a **symlink** to the repo copy so there is a single source of truth; if kept as separate files, keep them in sync and say so in the skill.
- Frontmatter must include `name` and `description` (Claude Code discovery). The repo copy may also carry OKF fields (`type: skill`, `title`, `tags`, `status`, `timestamp`); use `templates/skill-template.md`.

## Guardrails (why this stays controlled)

1. **Git is the control plane.** Every skill change is a commit — diffable, reviewable, and revertible by the owner. Nothing self-modifies invisibly. This is precisely what unbounded self-writing agents lack.
2. **Write real, not speculative.** One skill = one procedure the agent actually performs. Don't pre-write skills for things that haven't happened.
3. **Scope to your own repo.** An agent writes skills only in its own `kb-agent-<role>-<name>`. Cross-agent or privileged procedures live in the repo of the agent that owns them (e.g. agent lifecycle → the root-agent's `manage-agents` skill).
4. **Prune and mark, don't accrete.** Review skills periodically; when a procedure changes, update the skill; when it's obsolete, mark it **superseded** in place and archive rather than deleting silently.
5. **Keep them lean.** A skill points to `shared/runbooks/*` for deep detail instead of duplicating it; it sequences the job and records the gotchas.

## Approval

Authoring and updating skills in an agent's own repo is an internal-KB action allowed without approval (see `approval-policy.md`). Deleting a skill, or any skill with external side effects, follows the approval policy. The owner can veto or roll back any skill via git history at any time.
