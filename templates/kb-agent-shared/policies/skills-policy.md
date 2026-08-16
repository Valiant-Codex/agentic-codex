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
timestamp: 2026-08-07T00:00:00Z
---
# Skills Policy

## Purpose

Agents may **write and maintain their own skills** — reusable procedures they perform repeatedly. This is a deliberate capability (an agent that codifies its own know-how gets better over time), kept **controlled** so it never drifts into an unreviewable, sprawling mess. The control is structural: git, scope, a template, and pruning. The executable companion to this policy is the `skillify` skill.

## Model — config as code

> **This file is the single source of truth for skill format, naming, lifecycle and guardrails.**
> `skillify` executes it, `templates/skill-template.md` is its blank skeleton, and each agent's
> `skills/README.md` is an index only. If any of them states a rule this file does not, that is the
> bug — fix it here and let the others point.

### What the runtime actually enforces (verified, Claude Code 2.1.224)

- A skill **must** be a directory (or a symlink to one) containing `SKILL.md`. Plain `.md` files in
  `skills/` are discarded before they are even read — so a bare `foo.md` is not a skill, it is invisible.
- **The invocable name is the directory name**, not frontmatter `name`. `name` and `description` are
  both *optional* to the runtime: `name` is a display label, and a missing `description` falls back to
  text derived from the body.
- `SKILL.md` above **128 KB** is skipped.
- Symlinks are honoured at `~/.claude/skills` (this is what makes fleet-common skills work).

Our own rules are stricter than the runtime's on purpose: we require `name` and `description`, and
require `name` to equal the directory name. The runtime does not read that field — **we** do, and the
divergence check enforces it, so a rename can never leave the two disagreeing.


- A skill's **canonical copy** is a **folder** in the agent's own repo at `<agent-repo>/skills/<name>/`, whose entry point is `SKILL.md` (Anthropic Agent Skills format), versioned in git. Optional `references/` (deep detail, loaded on demand) and `scripts/` (executable helpers; only their output enters context) subdirectories provide progressive disclosure. Keep the `SKILL.md` body under ~5k tokens.
- **Fleet-common skills** — used by all agents (e.g. `skillify`, `agent-audit`) — live once in `kb-agent-shared/skills/<name>/` and are symlinked into each agent's tree (`ln -s ../shared/skills/<name> skills/<name>`): one canonical copy, propagated by kb-sync, no duplication.
- **Registration is automatic.** `~/.claude/skills` is a **whole-directory symlink** to the agent's repo `skills/`, so any `skills/<name>/SKILL.md` — including symlinked fleet-common skills — is harness-discoverable with zero per-skill setup. (This replaces the old per-skill `~/.claude/skills/<name>/SKILL.md` symlink model.)
- **Frontmatter** must include `name` and `description` (harness discovery; `description` states *what* the skill does **and when** to use it). The repo copy also carries OKF fields (`type: skill`, `title`, `tags`, `status`, `timestamp`). Use `templates/skill-template.md` as the blank skeleton.

## Guardrails (why this stays controlled)

1. **Git is the control plane.** Every skill change is a commit — diffable, reviewable, and revertible by <OWNER>. Nothing self-modifies invisibly. This is precisely what unbounded self-writing agents lack.
2. **Write real, not speculative.** One skill = one procedure the agent actually performs. Don't pre-write skills for things that haven't happened.
3. **Scope correctly.** An agent writes agent-specific skills only in its own `kb-agent-<role>-<name>`. Genuinely fleet-common procedures go in `kb-agent-shared/skills/` (symlinked into each agent). Cross-agent or privileged procedures live in the repo of the agent that owns them (e.g. agent lifecycle → the root agent's `manage-agents`).
4. **Prune and mark, don't accrete.** Review skills periodically; when a procedure changes, update the skill; when it's obsolete, mark it **superseded** in place with a banner at the top pointing at the replacement, or **delete it** — git holds the reasoning and the history. Do not park it in a parallel `skills-archive/` tree: the reference deployment tried that and the archived skill accumulated six live references, three of them with broken paths, while still being listed as active in its own registry. An archive directory is a second place to keep current.
5. **Keep them lean, and keep the depth inside the skill.** A `SKILL.md` sequences the job and records the gotchas; deep detail goes in that skill's own `references/` file, loaded only when the body points at it. Do **not** push it into a separate `runbooks/` tree: the runtime advertises skills by itself and advertises nothing else, so a runbook is reachable only if some already-loaded text happens to name its path. (The reference deployment carried a `runbooks/` directory for weeks in which exactly one of nine skills ever pointed at one.)
6. **No third-party skills in `shared/skills/`, ever.** Only skills we wrote. A `SKILL.md` is not inert data — it is instructions a model will follow — and `shared/skills/` reaches **every** agent's context, including the privileged one, through the unreviewed 15-minute kb-sync channel. A third-party skill installed there is third-party instructions injected into root's session. Externally-sourced skills, if ever adopted, are installed **per-agent** as a reviewed, pinned, vendored copy in that agent's own repo — never in the shared layer, never as a live fetch. Measured basis for the caution: of 3,984 public skills scanned in February 2026, 36.8% carried at least one vulnerability and 76 were confirmed malicious. See `decisions/2026-07-29-agent-skills-evaluated-build-ours-not-theirs.md`.
7. **The same applies to MCP tools, with more force.** Two servers exposing identically-named tools is the same shadowing problem in a layer you did not author: on the reference deployment one agent ran an Atlassian MCP whose 31 tools were a *strict subset* of a connector's 40. And unlike skills, every MCP tool **name** sits in the always-on context at ~16 tokens each, so a large unused server is a standing tax — see `docs/context-budget.md` and the `mcp-park` pattern.
8. **Fewer, distinct skills beat more.** Skill *selection* — not context length — is the dominant failure mode: with a large library the model picks the wrong skill or none at all ("shadowing"). The measured shape is a cliff, not a slope: 1–3 relevant skills help substantially more than 4+, and semantically overlapping skills degrade selection badly even at a constant count. So the question at review time is never "what should we add" but "which of these would still earn its place against a no-skill baseline". Prune accordingly.

## Lifecycle

`status:` is one of **`active`**, **`draft`**, **`superseded`**, **`archived`** — the same four values
the OKF standard defines in `skills/knowledge-governance-workflow`. (Earlier copies of `skillify`,
`skill-template.md` and the public `docs/skills.md` listed only three, omitting `draft`, while a live
skill shipped as `draft`. Four is correct; `draft` means written but not yet trusted to run.)

## Approval

Authoring and updating skills in an agent's own repo is an internal-KB action allowed without approval (see `approval-policy.md`). Deleting a skill, or any skill with external side effects, follows the approval policy. <OWNER> can veto or roll back any skill via git history at any time.
