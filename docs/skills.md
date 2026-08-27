<!-- title: Skills — folder-per-skill, auto-registered, self-authored under review -->
# Skills

A **skill** is a reusable procedure an agent performs repeatedly, stored in its brain as versioned
Markdown. Skills are how an agent codifies its own know-how so it gets better over time — kept
controlled so it never drifts into an unreviewable sprawl. The control is structural: Git, scope, a
template, and pruning.

## Format (Anthropic Agent Skills)

Each skill is a **folder** whose entry point is `SKILL.md`:

```
skills/<name>/
├── SKILL.md          # frontmatter + body (< ~5k tokens)
├── references/       # optional: deep detail, loaded only when the body points to it
└── scripts/          # optional: executable helpers; only their OUTPUT enters context
```

Frontmatter merges harness fields with your own knowledge-format fields:

```yaml
name: <lowercase-hyphens, <=64 chars>      # harness discovery
description: <what it does AND when to use it>   # the harness matches on this — make it trigger-rich
type: skill
title: <Human Title>
status: active            # active | draft | superseded | archived
```

**Progressive disclosure**: the tiny `name`+`description` is always in context; the body loads only when
triggered; `references/`/`scripts/` load only when referenced. Keep the body lean and point to
its own `references/` for depth instead of duplicating it.

## Registration (automatic)

The runtime's skills directory (`~/.claude/skills` for Claude Code) is a **whole-directory symlink** to
the agent's repo `skills/`, so any new `skills/<name>/SKILL.md` is discovered with **zero per-skill
setup**. Retire a skill with a `status: superseded` banner pointing at its replacement, or delete it — git
holds the body. There is no archive directory: a parallel tree is a second place to keep current.

## Fleet-common skills (single source, no duplication)

Skills every agent needs live **once** in `kb-agent-shared/skills/<name>/` and are symlinked into each
agent (`ln -s ../shared/skills/<name> skills/<name>`): one canonical copy, propagated by the sync timer,
discovered through the whole-dir symlink. Six ship with the framework — each one kept because it
proved useful in the reference deployment, not because it seemed like a good idea:

- **`skillify`** — the executable companion to `policies/skills-policy.md`: how to author, update, and
  retire a skill in this format, including the naming taxonomy, the lifecycle, and how to check a
  skill actually helps — a with/without sub-agent comparison, because a skill that reads well and
  changes nothing is the one a reviewer keeps by default.
- **`agent-audit`** — an interactive, human-in-the-loop tune-up of one agent's brain (its `CLAUDE.md`
  contract, skills, memory): the periodic sweep where accumulated experience becomes durable change
  (see [`memory.md`](memory.md)).
- **`advisor-review`** — how to get a genuinely independent second opinion out of a sub-agent: make it
  argue *against* you, constrain it (read-only, no credential hunting — a real incident, described in
  the skill, is why), and re-verify its load-bearing claims before you act on them.
- **`decision-loop`** — a sparring procedure for strategic calls: strawman rather than interrogate,
  work blocks in dependency order, falsify the load-bearing assumption early, record the outcome where
  it belongs instead of leaving it in a chat.
- **`knowledge-governance-workflow`** — the frontmatter and hygiene rules that keep a brain
  machine-readable and lean.
- **`topic-management`** — how an agent manages *its own* Remote Control sessions through the
  root-owned `claude-topic` wrapper, which scopes itself to the caller's brain repo by Unix username.
  The capability exists the moment the wrapper is installed; what this skill adds is the knowledge that
  it does, and the judgement around it — that the wrapper has **no self-guard** (stopping the topic you
  are running in kills the process executing the command, so check `/proc/self/cgroup` first), that
  `remove` is a human-confirm gate while `stop` is usually what was meant, and that every new topic is
  a permanently-on session someone pays for continuously.

## The guardrail

**Git is the control plane.** Every skill change is a diffable, revertible commit the owner can veto —
nothing self-modifies invisibly. Agents may author skills in their own repo (and genuinely fleet-common
ones in `shared/skills/`), but always in a session with the owner present — **no scheduled job ever
changes a skill or an identity file.** This is what separates "improves over time" from "drifts out of
control."

> **A caveat worth knowing:** a `SKILL.md` is *instructions a model will follow*, not inert data. If your
> shared layer is auto-synced (see [`config-model.md`](config-model.md)) then a skill written in the
> shared repo reaches every agent's context without review. Either keep fleet-common skills in the
> privileged agent's own repo, or accept that the shared repo's write access is as trusted as the agents
> that load it.
