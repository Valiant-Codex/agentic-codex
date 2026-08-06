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
  to which agent), *Human-Confirm Gates*, *Autonomous-OK*, *Trust / Threat Model*, *Source Of Truth*.
  **No bootstrap section** — the read order lives in `CLAUDE.md`, once. See the one-place rule below.
- **CLAUDE.md**: **the** bootstrap — `provision-agent` symlinks it to `~/CLAUDE.md`, and since the topic
  unit sets `WorkingDirectory=%h` this is the only bootstrap the running agent loads. Keep it thin: it
  points at SOUL + OPERATING + `shared/owner-profile.md` and states the always-on untrusted-content
  rule, without restating their content. **Use absolute `~/github/<org>/<brain>/…` paths** —
  repo-relative paths do not resolve from cwd=`~`, which is the trap that makes a two-file layout
  fragile.
  (An optional `AGENTS.md` sibling adapter serves non-Claude harnesses; add it only when a second
  framework is actually in use.)
- Who <OWNER> and the fleet are lives once, fleet-wide, in `shared/owner-profile.md` — never
  duplicated per agent.

### The one-place rule

A fact belongs in the always-loaded layer **only if the agent cannot know to go looking for it.**
Everything else is retrieved on demand, and is stated exactly once.

Always-on, therefore stated verbatim in `CLAUDE.md`: the rules that must bind *before* the agent knows
a policy exists — untrusted-content, and any hard domain rule such as data residency — the trigger
sentence that sends it to `approval-policy.md`, and the delegation map, because an agent cannot lazily
retrieve the knowledge that a task belongs to someone else.

Retrieved on demand, therefore stated once in its own file and merely pointed at: everything else.

This is a correctness rule, not a budget. Measured on the reference fleet, 2026-08-06, harness
2.1.223: a fresh session loads 19.6k of 1.0M (2%), with MCP schemas deferred. There was nothing
meaningful to reclaim, so do not trim to save tokens. What a duplicated read order costs is not tokens
but truth — copies drift, each one looks authoritative, and that fleet carried two standing bugs from
exactly this: repo-relative paths that do not resolve from cwd=`~`, and a step ordering a read of
`CLAUDE.md`, which the harness has already loaded before the agent reads anything.

The same test applies below the read order. A hard rule written verbatim in `CLAUDE.md` and restated
at the head of its `OPERATING.md` section is two copies of one rule; keep the binding statement in
`CLAUDE.md` and let the section carry only what `CLAUDE.md` does not — the elaboration, the tables,
the rationale.

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
