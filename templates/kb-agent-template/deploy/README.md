---
type: directory-readme
title: Deploy — the runtime wiring, declared in git
description: What this agent needs in order to exist on a machine — its topic sessions and its permission settings. Read by provision-agent; this is the directory that makes clone-and-run possible.
tags:
- directory-readme
- deploy
- provisioning
- portability
status: active
timestamp: 2026-08-07T00:00:00Z
---
# Deploy — the runtime wiring

Everything else in this repo is what the agent *knows*. This directory is what the agent *needs in
order to run* — declared in git so a machine can be rebuilt from it without anyone remembering.

`provision-agent` reads it. Nothing here is loaded into the agent's context.

| File | What it does |
|---|---|
| `topics.tsv` | The agent's Remote Control sessions: one row per topic, `key<TAB>Display Name`. `provision-agent` enables a `claude-topic@<key>` systemd user unit for each. `claude-topic new/remove` maintain this file and commit it themselves. |
| `claude-settings.json` | The agent's Claude Code permissions and preferences. Installed as a **real 0600 copy** at `~/.claude/settings.json` — never symlinked, because the auto-sync must not be able to change what an agent is permitted to do. Edit it here, then re-run `provision-agent` (or reinstall it) to apply. |
| *(optional)* `output-styles/<agent>.md` | The agent's Claude Code output style (voice/register — frontmatter `name`/`description`/`keep-coding-instructions`, then plain-Markdown instructions). **Symlinked**, not copied, at `~/.claude/output-styles/<agent>.md`: unlike `claude-settings.json` it carries no permission grant, so there is no reason to gate it behind an explicit re-provisioning step. Set `outputStyle` to `<agent>` in `claude-settings.json` to activate it. |
| *(optional)* `*.service` | A systemd **user** unit this agent needs beyond its topics — e.g. an MCP server it runs for itself. |

## Why this exists as a directory rather than as documentation

This is the portability boundary. The claim "clone the repo, restore one secret, run one command, and
the agent is back" is true only because the answers to *which sessions?* and *what is it allowed to do?*
are files rather than institutional memory. A brain without `deploy/` is a set of notes; a brain with it
is a deployable agent.

It is also the **security boundary**. `claude-settings.json` is the permission allowlist, and it is
deliberately installed by the explicit, human-run provisioning step — not by the 15-minute auto-sync
that refreshes the rest of the repo. Inert data syncs automatically; anything that grants capability
requires a person to run a command. See ``docs/config-model.md``.

## What does **not** belong here

Secrets, of any kind. Tokens, API keys and MCP credentials are restored out-of-band into files outside
git (`~/.config/<agent>/secrets.env`, `~/.claude.json`). `.mcp.json` at the repo root references them as
`${ENV}` placeholders and never contains a value.
