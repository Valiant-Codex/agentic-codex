---
type: template
title: Agent Template
description: How to scaffold a new agent and what a good agent brain contains (v2 — SOUL/OPERATING split, folder-per-skill, shared owner-profile).
tags:
- template
- agent
- okf
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Agent Template

New agents are scaffolded from your own `kb-agent-template` repo (copied from this framework's
`templates/kb-agent-template/`) and provisioned onto
a box with `infra/scripts/provision-agent` (which clones the brain + `kb-agent-shared` as a sibling and
wires the `shared/` symlink — no submodule commands). This document is the checklist for what a brain
should contain and how the pieces fit. For the *why* and boundary contract of a new agent, write a
decision in `decisions/` first — state what it owns, what it must delegate, and why it deserves its own
Unix user and token.

## Repo shape (`kb-agent-<role>-<name>`)

```
SOUL.md              ← durable identity: who the agent is, voice, principles (loaded every session)
OPERATING.md         ← operating contract: scope/boundaries, gates, autonomous-OK, bootstrap
CLAUDE.md            ← thin Claude Code adapter → loads SOUL + OPERATING + shared/owner-profile
context/             ← stable background specific to this agent (optional; owner facts live in shared)
memory/              ← durable memory (distilled-memory.md + episodic/); see policies/memory-policy.md
tools/               ← tool/MCP registry (README.md); secrets are ${ENV} placeholders, never committed
skills/              ← folder-per-skill (<name>/SKILL.md); fleet-common ones symlink shared/skills/*
deploy/              ← topics.tsv, claude-settings.json (+ any per-agent systemd user units)
.mcp.json            ← MCP servers with ${ENV} placeholders — no secrets
shared -> ../kb-agent-shared   ← committed symlink to the sibling clone (global governance)
```

## Identity: SOUL.md + OPERATING.md

Identity is split so durable "who you are" changes rarely and separately from mutable "what you do":

- **SOUL.md** (durable, human-owned, ~300–400 words): *Identity* (one line + a naming framing),
  *Voice* (a distinct, recognizable register — each agent differs), *Principles* (how it embodies its
  role). Loaded every session.
- **OPERATING.md** (the operating contract): *Scope / Boundaries* (what it owns, and what it delegates
  to which agent), *Human-Confirm Gates*, *Autonomous-OK*, *Trust / Threat Model*, *Source Of Truth*,
  *Bootstrap Contract*.
- **CLAUDE.md**: a thin (~15-line) adapter that loads SOUL + OPERATING + `shared/owner-profile.md` and
  states the always-on untrusted-content rule. (It serves
  non-Claude harnesses; add it only when a second framework is actually in use.)
- Who <OWNER> and the fleet are lives once, fleet-wide, in `shared/owner-profile.md` — never
  duplicated per agent.

## Skills

Folder-per-skill: `skills/<name>/SKILL.md` with `name` + `description` frontmatter (the `description`
states *what* and *when*). `~/.claude/skills` is a whole-directory symlink to `skills/`, so new skills
are auto-discovered. Fleet-common skills live once in `kb-agent-shared/skills/` and are symlinked in
(`ln -s ../shared/skills/<name> skills/<name>`). See `policies/skills-policy.md` and the `skillify` skill.

## Cross-cutting rules

- **Boundaries:** keep each agent in its lane; name the agent that owns work outside scope. Cross-agent
  handoffs are relayed by the owner in chat (the sending agent writes the brief as Markdown, the owner
  pastes it into the receiving agent's session) — there is no handoff directory.
- **Access:** one agent = one Unix user + one GitHub bot account; least-privilege per
  `policies/github-access-policy.md` and `runbooks/provision-agent-github-access.md`.
- **Governance freshness:** `shared/` stays current via kb-sync (no submodule commands). Runtime and
  portability: `runbooks/agent-ops-and-portability.md`.
