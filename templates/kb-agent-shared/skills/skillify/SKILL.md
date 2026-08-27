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
timestamp: 2026-08-27T00:00:00Z
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

Two levels. The first checks the shape; the second checks whether the skill is worth its place.

**Shape — every time:**

- `name` + `description` present; `name` equals the folder name; lowercase-hyphens, <=64 chars, no
  reserved words.
- Body under ~5k tokens; deep detail lives in `references/`.
- After the skills dir changes, the skill appears in the harness skill list (whole-dir symlink). If it
  doesn't, check the folder name and the frontmatter.

**Effectiveness — for every new skill, and at every review:**

`skills-policy.md` says the review question is "would this still earn its place against a no-skill
baseline". This is how you answer it instead of guessing.

1. Write 2–3 realistic test prompts — what <OWNER> would actually type, not a paraphrase of your own
   `description`. A prompt written from the description tests the description, not the skill.
2. For each prompt, spawn **two** sub-agents **in the same turn**: one given the skill's path, one
   given no skill at all. Same prompt, same inputs. Write outputs under the session scratchpad,
   `<scratchpad>/skill-eval/<name>/<prompt-id>/{with,without}/`. Launch them together — staggering
   them invites you to read the first result and pre-judge the second.
3. Compare, and name the outcome. **The skill wins** — keep it. **A tie** — the model already knew,
   or the skill is too generic to add anything; either way it is a candidate for deletion, not for
   more prose. **The skill loses** — almost always because it steers wrong, not because it is missing
   a section; cut the steer rather than adding a caveat.
4. **Triggering is a separate test from quality.** A good skill that never fires is worth nothing.
   Take a prompt that *should* reach for it and check whether it does, without naming the skill. If
   it doesn't fire, the defect is in the `description`, not the body — that is the field the harness
   matches on.

The rule underneath all of it: **a run with the skill proves nothing without the run without it.**
A plausible-looking output is not evidence the skill caused it.

When updating an existing skill, the baseline is the *old version*, not no-skill: snapshot it before
editing and point the second sub-agent at the snapshot.

## Related

- `shared/policies/skills-policy.md` — the policy this skill enacts.
- `shared/templates/skill-template.md` — a blank v2 skeleton to copy.
- The with/without evaluation above is borrowed from Anthropic's official `skill-creator` plugin,
  **deliberately absorbed rather than installed** (2026-08-27). Installing it would have put
  third-party instructions beside ours in a root-privileged session, and left two skills competing on
  the same intent — the shadowing `skills-policy.md` guardrail 8 warns about. Do not re-propose
  installing it; take further ideas from it by hand, the same way.
