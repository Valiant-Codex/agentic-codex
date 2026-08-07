---
name: skillify
description: Author, update, or retire a <ORG> agent skill in the v2 format (folder + SKILL.md, name+description frontmatter, progressive disclosure, lifecycle). Use after you've repeated a procedure worth codifying, when a skill needs updating, or when adding/retiring one.
type: skill
title: Skillify — author and maintain skills
tags:
- skill
- meta
- authoring
- lifecycle
- fleet
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Skillify

The executable companion to `shared/policies/skills-policy.md`: how to write, update, and retire a
skill so the fleet's skills stay discoverable, lean, and low-drift. This is a **fleet-common** skill —
it lives once in `shared/skills/skillify/` and is symlinked into each agent's `skills/`.

## When to use

- You've performed a procedure two or three times and it's worth codifying.
- An existing skill is out of date, or its steps changed.
- You're adding a new capability, or retiring an obsolete one.

Do **not** pre-write speculative skills for things that haven't happened. One skill = one procedure you
actually perform.

## Format (v2 — Anthropic Agent Skills)

A skill is a **folder** `skills/<name>/` whose entry point is `SKILL.md`:

```
skills/<name>/
├── SKILL.md          # frontmatter + body (<5k tokens)
├── references/       # optional: deep detail, loaded only when the body points to it
└── scripts/          # optional: executable helpers; only their OUTPUT enters context
```

Frontmatter merges harness fields with OKF:

```yaml
name: <lowercase-hyphens, <=64 chars, must not contain "claude" or "anthropic">
description: <what it does AND when to use it — <=1024 chars; the harness matches on this>
type: skill
title: <Human Title>
tags: [ ... ]
status: active            # active | draft | superseded | archived
timestamp: <ISO-8601>
```

- **Body under ~5k tokens.** Sequence the job and capture the gotchas; push deep detail into
  `references/` and long/deterministic steps into `scripts/`. Point to this skill's own `references/*` instead of
  duplicating them.
- **`description` is the most important field** — make it trigger-rich: state both *what* it does and
  *when* to reach for it.

## Naming taxonomy

- **capability** (action-based): `dokploy-service-mgmt`, `notion-authoring`
- **behavioral** (layered method): `<domain>-<method>`
- **outcome / domain**: `agent-audit`, `patch-management`

## Registration (automatic)

`~/.claude/skills` is a whole-directory symlink to your repo's `skills/`, so any new
`skills/<name>/SKILL.md` is discovered with zero setup. **Fleet-common skills** live once in
`shared/skills/<name>/` and are symlinked into each agent's tree:
`ln -s ../shared/skills/<name> skills/<name>`.

## Procedure

- **Create:** pick a name (taxonomy above) → `mkdir skills/<name>` → write `SKILL.md` (frontmatter +
  lean body) → add `references/`/`scripts/` only if genuinely needed → validate (below) → commit. The
  commit **is** the review gate; <OWNER> can veto or roll back via git history.
- **Update:** edit in place and commit. If behavior changed materially, say so in the commit.
- **Retire:** mark `status: superseded` with a top banner pointing to the replacement (keep it in place
  while anything still references it), or **delete it** — git holds the body and the reasoning. Do not
  park it in a parallel `skills-archive/` tree: it becomes a second place to keep current, and the
  reference deployment's archive grew six live references to a skill its own registry still called active.

## Guardrails

- **Git is the control plane.** Every change is a diffable, revertible commit — nothing self-modifies
  invisibly. This is what keeps self-authored skills from drifting into an unreviewable mess.
- **Write real, not speculative.**
- **Scope correctly.** Your own repo for agent-specific skills; `shared/skills/` only for genuinely
  fleet-common ones; privileged/cross-agent procedures belong to the agent that owns them.
- **Lean, not sprawling.** Push depth into `references/`; prune and mark superseded; don't accrete.

## Validation

- `name` + `description` present; `name` is lowercase-hyphens, <=64 chars, no reserved words.
- Body under ~5k tokens; deep detail lives in `references/`.
- After the skills dir changes, the skill appears in the harness skill list (whole-dir symlink). If it
  doesn't, check the folder name and the frontmatter.

## Related

- `shared/policies/skills-policy.md` — the policy this skill enacts.
- `shared/templates/skill-template.md` — a blank v2 skeleton to copy.
