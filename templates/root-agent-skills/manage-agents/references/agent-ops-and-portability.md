---
type: reference
title: Agent Operations & Portability
description: How <ORG> agents run on the VPS with total versioning, resilience, and portability — config as code, session supervision, and fresh-VPS recovery.
tags:
- reference
- portability
- resilience
- systemd
- claude-code
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Agent Operations & Portability

Goal: **`git clone` + one secret + a few commands = the same agent, live, on any VPS.** No hidden
state on the box. This runbook is the source of truth for how agents are versioned, kept alive, and
recovered. The **root-agent** (`kb-agent-ops-<name>`) is the reference implementation; the same shape
applies to every agent repo.

## Principle: the agent repo is the single source of truth

Everything about an agent that is not a secret or volatile runtime state lives in its repo
(`kb-agent-<role>-<name>`) and is symlinked into the home directory. Editing the repo file edits the
live config; cloning the repo restores it.

| Concern | Repo path (versioned) | Home location (symlink → repo) |
|---|---|---|
| Identity / behavior | `CLAUDE.md` | — (read from the repo) |
| Runtime bootstrap | `CLAUDE.md` (repo root — the single bootstrap; absolute `~/github/...` paths so it resolves from cwd=`~`) | `~/CLAUDE.md` |
| Claude Code settings | `deploy/claude-settings.json` (curated prefs + allowlist; portable) | `~/.claude/settings.json` — **real copy** installed by `provision-agent`, not a symlink |
| MCP servers | the runtime's user-scoped config (holds credentials, not in git). A repo `.mcp.json` is project-scope for the agent's **cwd** — so it is not read when the session runs with `WorkingDirectory=%h`. | the box |
| Skills / tools / memory | `skills/`, `tools/`, `memory/` | — |
| Session supervision (infra) | `infra`: `bin/claude-topic`, `systemd/claude-topic@.service` → root-owned `/usr/local/bin/claude-topic` + per-user unit copies | `~/.local/bin/claude-topic` (→ `/usr/local/bin/claude-topic`), `~/.config/systemd/user/claude-topic@.service` (real copy) |
| Topic registry (per-agent) | agent repo `deploy/topics.tsv` (`key -> Display Name`) | — |
| Shared governance | `shared/` (symlink → sibling clone `../kb-agent-shared`) | — |

**Not versioned** (by design): secrets (tokens/keys) and volatile runtime state (`~/.claude.json`
account/session state, `~/.config/agent/topics.state` session IDs, `~/.claude/settings.local.json`
auto-accumulated per-session approvals).

## Secrets

Secrets never go in git. Today they live where the runtime reads them: MCP tokens in `~/.claude.json`
(mode 600) and/or a `~/.config/<agent>/secrets.env` (mode 600) referenced by `${ENV}` in `.mcp.json`
and sourced by the systemd unit (`EnvironmentFile=`). The privileged root-agent can update any
secret on <OWNER>'s behalf from chat — <OWNER> does not need shell access.

**Secret backup/portability.** For a fresh-VPS restore you need the secrets somewhere restorable without
pasting each one by hand. The recommended approach is a **self-hosted secret store — Vaultwarden on
Dokploy — backed up off the box** (see the Agentic Codex docs (`secrets.md`)); `age`-encrypted-in-git
was considered and rejected as too heavy. Either way, "restore secrets" is an out-of-band step in the
recovery procedure below.

## Session resilience (survive crash + reboot)

A topic is one `claude --remote-control '<Name>'` process. Remote Control gives access from any device,
so there is **no tmux** — a **systemd user service** (`claude-topic@<key>`) supervises the process
directly. This is the single, robust, repeatable model:

- `Restart=always` → survives crashes.
- `loginctl enable-linger <agent>` → the user's systemd runs at boot without login → survives reboot.
- Unit + registry are versioned in the repo → portable.

The `claude-topic` wrapper is **root-owned** at `/usr/local/bin/claude-topic` (canonical source: `infra/bin/claude-topic`),
with each agent's `~/.local/bin/claude-topic` symlinked to it — so no agent can rewrite what another agent
executes (a fleet security hardening measure). The command is named after the
unit it drives (`claude-topic` → `claude-topic@<key>.service` → `topics.tsv`); it was renamed from the
bare `topic` (the compatibility symlink was later removed). It is systemd-native:

```
claude-topic list                     # all topics: key, service state, sessionId, display name
claude-topic urls                     # every topic's Remote Control URL (test them BY HAND)
claude-topic new <key> "<Name>"       # register + enable--now the service, capture the new sessionId
claude-topic restart <key>            # systemctl restart (resumes the same sessionId)
claude-topic restart --new <key>      # explicit opt-in: start a FRESH conversation
claude-topic rotate <key>             # abandon the conversation, mint a NEW bridge id — the fix when a
                                     #   topic is active but the app says "session can't be found"
claude-topic rotate-all [--dry-run]   # rotate every enabled topic (what the boot unit runs)
claude-topic stop <key>               # stop + disable the service (sessionId is kept)
claude-topic remove <key>             # unregister for good: topics.tsv + state + unit (use this, not just `stop`,
                                     #   when a topic is deleted — else agentic-monitor keeps reporting it)
claude-topic status <key>             # service state + stored/live sessionId + bridge URL
claude-topic remember <key>           # record a running topic's live sessionId to state
claude-topic run <key>                # foreground launch (invoked by the systemd unit)
```

**`active` does not mean reachable.** The Remote Control bridge is server-side state that dies with
the conversation, and nothing on the box can observe it: a topic can be `active`, `enabled`, under
`Restart=always`, with a valid sessionId — and still answer "session can't be found" in the apps.
`restart` does not help, because resuming re-announces the same dead bridge; only `rotate` mints a
new one. `urls` prints what the session *announced*, so opening one is the only proof.

Because the failure is invisible locally, reachability after a reboot is not checked but *acted on*:
`claude-topic-rotate-on-boot.service` runs `rotate-all` once per boot and writes a digest of the
abandoned sessions and new URLs to `~/.config/agent/last-boot-rotation.tsv`. The cost is deliberate —
**every topic starts a fresh conversation after a reboot**, so nothing that matters should live only
in a conversation.

The wrapper fails fast rather than letting systemd fail-loop a topic: unknown keys are
rejected before `systemctl` is called (`claude-topic@<typo>` is a valid template instance
and would otherwise start, fail, and burn through `StartLimitBurst`), and a restart is
refused when there is nothing to resume — no stored sessionId, or one whose transcript is
missing, which makes `claude --resume` hard-fail. Both refusals point at `--new`, the only
way to deliberately start a blank conversation. `restart` also re-enables a unit left
disabled by `stop`, so a stopped-then-restarted topic still survives a reboot.

`deploy/topics.tsv` (versioned) maps `key -> Display Name`. `~/.config/agent/topics.state` (runtime)
maps `key -> sessionId` so a restart resumes the same conversation. `claude-topic run` joins them and execs
`claude --resume <id> --remote-control '<Name>'`.

### Unit gotchas (baked into the shared unit / setup — documented so they don't get lost)

- **pty:** claude is a TUI — without a terminal it drops into `--print` mode and exits. The unit runs
  it inside a pty via `/usr/bin/script -qfec "…" /dev/null`, staying foreground so systemd supervises
  it directly.
- **PATH:** systemd's default PATH lacks `~/.local/bin` (claude, claude-topic) and `~/.local/node/bin` (npx
  for local MCP servers); the unit sets `Environment=PATH=…`.
- **trust:** a headless claude blocks on the "trust this folder?" prompt. Pre-accept it once per agent
  home: set `projects["/home/<agent>"].hasTrustDialogAccepted = true` in `~/.claude.json`. Also set
  top-level `hasCompletedOnboarding = true` (and `projects[...].hasCompletedProjectOnboarding = true`)
  if the account was authed via a token/device flow that skipped the onboarding TUI.
- **first run:** `claude-topic run` must tolerate a missing `~/.config/agent/topics.state` (no sessionId yet).
  `state_sid`/`reg_name` end with `|| true` so `set -e` doesn't kill the launch before `claude` execs —
  otherwise a brand-new agent's first `claude-topic new` fails silently (~31 ms exit, service flaps `activating`).

### Provision systemd for an agent (once)

```bash
sudo ORG=<your-github-org> scripts/provision-agent <unix-user> <brain-repo>   # from the canonical infra clone
# then, per topic, as that agent:
claude-topic new <key> "<Display Name>"                                       # register + start under systemd
```

`provision-agent` installs the wrapper, the topic unit, linger, the topics, the roster entry and the
nightly memory mirror, and refuses to act on a clone that is dirty, unpushed, or would shrink the
fleet roster. This section used to list those steps as hand-run commands, beginning with
`sudo install … /usr/local/bin/claude-topic` — a root install of the binary **every** agent executes,
with none of those guards, addressed to the agent's own user. Do not reintroduce it: if provisioning
needs a new step, the step belongs in `provision-agent`, not in a runbook someone has to remember.

## Fresh-VPS recovery / migration (disaster recovery)

The bring-up is a single privileged step — `infra/scripts/provision-agent` — not a manual checklist. See
the Agentic Codex docs (`config-model.md`) for why (git = portable source of truth; provisioning = apply + security
boundary; kb-sync auto-syncs only inert data).

Bring an agent up on a new box:

1. **Prereqs (not in git):** create the Unix user (sudo only for the root-agent — see the approval
   policy); install Claude Code and, if the agent uses `npx`-based MCP servers,
   Node into `~/.local/node`; **restore the agent's secrets** — its `gh` token to
   `~/.config/gh/hosts.yml` (mode 600) with git identity set, and any MCP secrets in `~/.claude.json`.
2. **Provision:** as the root-agent, `provision-agent <unix-user> <brain-repo>`. It clones the brain +
   `kb-agent-shared` sibling, wires the symlinks (`shared`, `~/CLAUDE.md`, `~/.local/bin/claude-topic`),
   installs `/usr/local/bin/claude-topic` + the per-user systemd unit + `~/.claude/settings.json` (real copy
   from `deploy/claude-settings.json`), enables linger, and enables/starts the topics in
   `deploy/topics.tsv`. Idempotent; fails fast if the `gh` token is missing.
3. **Verify:** `claude-topic list` shows the topics live with their sessionIds; each opens in claude.ai / the
   mobile app. (Topic `sessionId`s are per-machine runtime state, not carried — a brand-new box starts
   fresh sessions.)

## Related

- the Agentic Codex docs (`config-model.md`) — the runtime pattern (topics, restart mechanics, ownership) and the
  provisioning / portability / security boundary.
- the Agentic Codex docs (`monitoring.md`) — how topic sessions are monitored for liveness.
