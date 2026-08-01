---
type: runbook
title: Manage Agents (create / manage / decommission)
description: The full agent lifecycle on a VPS — GitHub bot + Unix user + runtime + systemd topic sessions, and clean removal. The privileged root-agent's procedure, leaning on provision-agent and the GitHub-access runbook.
tags:
- runbook
- agents
- provisioning
- lifecycle
- systemd
- remote-control
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Manage Agents

The privileged **root-agent** owns the full agent lifecycle on `<VPS_HOST>` (it has sudo; the other
agents do not). This is the actionable procedure; the deep model lives in
`runbooks/agent-ops-and-portability.md`, `runbooks/provision-agent-github-access.md`, and
`policies/github-access-policy.md`. Read those for detail; this runbook sequences the whole job and
captures the gotchas that actually bite.

## When to use

The owner asks to **add**, **reconfigure**, or **remove** an agent, or to manage an agent's topic
sessions. First confirm the agent is justified (does the task deserve its own identity, bot, and blast
radius? — see `docs/multi-agent-governance.md`) and record a decision if it's a new agent.

## Prerequisites

- The root-agent runs as its Unix user with sudo.
- **Collaborator invites need org-admin**, which the root-agent's bot token does NOT have. So GitHub
  account creation, org PAT policy, and inviting the bot to repos are the **owner's** steps. The
  root-agent does everything on-box and can **accept** invites via the new bot's own token.

---

## CREATE a new agent

Naming: role `<ROLE>`, short name `<AGENT>` (the Unix user), brain repo
`<BRAIN>` = `kb-agent-<ROLE>-<AGENT>`, GitHub bot `<bot-username>`.

### (a) Owner-side GitHub steps — see `runbooks/provision-agent-github-access.md`

The bot account, its email (Google Group `<AGENT>-agent@<your-domain>`), least-privilege repo access,
and the token all come from that runbook. Do it first: **do not duplicate those steps here.** Its
output is a working bot with a token to be restored on-box (step (c) fails fast without it).

⚠️ **Do the first-ever interactive Claude login now, before step (c) — not after.** As the new user:
`ssh <AGENT>@<VPS_HOST>` → `claude` → log in with the owner's account → quit. The order matters: if a
topic session first starts *unauthenticated*, it registers a sessionId that never gets a transcript on
disk, and every later `claude-topic restart` then correctly refuses to resume a conversation that does
not exist. Recovering costs a `claude-topic restart --new <key>` and loses nothing, but it is avoidable
noise — log in first.

### (a2) Seed the brain repo — `provision-agent` cannot invent it

**A brand-new brain repo is usually empty, and provisioning an empty repo produces a broken agent, quietly.**
`provision-agent` symlinks `~/CLAUDE.md → <BRAIN>/CLAUDE.md` (the brain's root bootstrap) and reads
`<BRAIN>/deploy/topics.tsv`; if the repo has neither, you get a **dangling** `~/CLAUDE.md` (the agent
boots with no bootstrap at all) and **zero topics enabled** (`[warn] no deploy/topics.tsv`).

So before step (c), scaffold the brain from `templates/kb-agent-template` and push it, with at minimum:

| Required | Why |
|---|---|
| `CLAUDE.md` (repo root) | the target of `~/CLAUDE.md`; without it the agent has no bootstrap |
| `deploy/topics.tsv` | the topic registry; without it no session is ever started |
| `deploy/claude-settings.json` | else `~/.claude/settings.json` is skipped (`[warn]`), leaving no allowlist |
| `SOUL.md` + `OPERATING.md` | what the bootstrap points at — a bootstrap naming missing files fails the divergence check |

Commit and push as the new bot (this is also the first real test that its token has write).

### (b) On-box: create the user and install the runtime

```bash
# 1. Unix user (NO sudo — only the root-agent has sudo) + linger so sessions survive reboot
sudo useradd -m -s /bin/bash -c "<AGENT>" <AGENT>
sudo loginctl enable-linger <AGENT>

# 2. Restore the bot token out-of-band (secret, never in git) — see the GitHub-access runbook step 6:
#    write ~<AGENT>/.config/gh/hosts.yml (mode 600), set git user.name/user.email from ~.

# 3. Runtime: install Node + Claude Code. If cloning from another agent for a deterministic, offline,
#    identical version:
#    NOTE: ~/.local/node is a SYMLINK — copy the DEREFERENCED real dir or you copy a dangling link.
sudo cp -a "$(readlink -f /home/<other-agent>/.local/node)" /home/<AGENT>/.local/node
sudo cp -a /home/<other-agent>/.local/share/claude /home/<AGENT>/.local/share/claude
sudo mkdir -p /home/<AGENT>/.local/bin
sudo ln -sf ../node/bin/node /home/<AGENT>/.local/bin/node
sudo ln -sf ../node/bin/npx  /home/<AGENT>/.local/bin/npx
sudo ln -sf ../node/bin/npm  /home/<AGENT>/.local/bin/npm
sudo ln -sf /home/<AGENT>/.local/share/claude/versions/<ver> /home/<AGENT>/.local/bin/claude
sudo chown -R <AGENT>:<AGENT> /home/<AGENT>/.local

# 4. Pre-accept trust + onboarding in ~/.claude.json (merge — preserve any existing login/auth!),
#    or a headless topic session exits in ~31 ms.
sudo -u <AGENT> -H python3 - <<'PY'
import json,os
p=os.path.expanduser('~/.claude.json'); d=json.load(open(p)) if os.path.exists(p) else {}
d['hasCompletedOnboarding']=True
pj=d.setdefault('projects',{}).setdefault(os.path.expanduser('~'),{})
pj['hasTrustDialogAccepted']=True; pj['hasCompletedProjectOnboarding']=True
json.dump(d,open(p,'w'))
PY
```

### (c) Apply the rest with `provision-agent` — do NOT do it by hand

From your deployed infra clone (`~/github/<ORG>/infra/scripts/`), run:

```bash
ORG=<ORG> provision-agent <AGENT> <BRAIN>
```

It is idempotent (safe to re-run) and does everything git carries: clones the brain + `kb-agent-shared`
sibling, wires the symlinks (`shared`, `~/CLAUDE.md`, `~/.local/bin/claude-topic`), installs the
root-owned `claude-topic` wrapper + per-user systemd unit, materializes `~/.claude/settings.json` as a
real copy, enables linger, and starts each topic in `deploy/topics.tsv`. It fails fast if the bot token
is missing (that is why (a)/(b) come first). **Do not re-do the clone/symlink/unit/topic steps
manually** — reference `provision-agent`.

`provision-agent` also symlinks every fleet-common skill from `shared/skills/` into the brain's
`skills/`, and prints them if they are untracked: **commit those symlinks** — they are part of the
brain's declared shape, and leaving them untracked keeps the repo dirty from day one (provisioning
deliberately never commits into an agent's own repo).

The agent is then auto-kept-current by `kb-sync`. Brain **content** (the SOUL.md and OPERATING.md identity,
tools, skills) is authored by the agent (or its designated maintainer), not by the root-agent.

### (d) Commit the roster change provision-agent made

`provision-agent` already appended `<AGENT>` to `infra/fleet-agents` and refreshed the installed
copy — do **not** edit the roster by hand. What remains is the git half: commit and push the infra
repo (the daily divergence check reports a dirty infra clone until you do).

## MANAGE sessions

Topic sessions are systemd user services fronted by the `claude-topic` wrapper. Run as the agent (the
provisioner already set PATH + `XDG_RUNTIME_DIR`/`DBUS_SESSION_BUS_ADDRESS` for its units):

`claude-topic list` · `claude-topic new <key> "<Name>"` · `claude-topic restart <key>` (resumes the same
sessionId) · `claude-topic stop <key>` · `claude-topic status <key>`.

**Any MCP or config change needs a `claude-topic restart`** — MCP loads at process start, so a running
session won't pick it up otherwise.

## REMOVE / decommission an agent  ⚠️ DESTRUCTIVE — human-confirm with the owner first

```bash
# 1. Stop + disable all its topic services, then linger
sudo -u <AGENT> -H bash -c 'export XDG_RUNTIME_DIR=/run/user/$(id -u); for s in $(systemctl --user list-units "claude-topic@*" --all --plain --no-legend | awk "{print \$1}"); do systemctl --user disable --now "$s"; done'
sudo loginctl disable-linger <AGENT>
# 2. Disable its nightly memory-mirror timer (a leftover one FAILS nightly after userdel and pages
#    the monitor every cycle until someone notices)
sudo systemctl disable --now memory-mirror@<AGENT>.timer
# 3. Back up the home if anything un-pushed matters, then remove the user
sudo tar czf /root/backup-<AGENT>-$(date +%F).tgz /home/<AGENT>   # keep until sure
sudo userdel -r <AGENT>
# 4. Remove <AGENT> from infra/fleet-agents, commit + push, and re-run install-host-services —
#    the divergence check reports a stale roster line (and a stale mirror timer) either way
```
5. GitHub (owner, or a token with org admin): **revoke the bot PAT**, remove the bot as collaborator or
   delete the bot account, and **archive the brain repo**.
4. **Record a decision** documenting the decommission.

## Gotchas (learned the hard way)

- **Node is a symlink** → copy the dereferenced real dir (`readlink -f`), else you copy a dangling link
  (`node: command not found` / a `Permission denied` into the source agent's home).
- **Onboarding**: a token/device login skips the onboarding TUI → set `hasCompletedOnboarding` + the
  per-project trust flags in `~/.claude.json`, or the headless topic exits in ~31 ms.
- **First-run `topics.state`**: the topic runner needs to tolerate a missing state file on a brand-new
  agent's first `claude-topic new`; older wrapper copies crash there.
- **cwd matters**: run `git config …` from the agent's `~` — git stats the cwd and fails on directories
  it can't read (e.g. another agent's home).
- **The root-agent can't invite collaborators** (its bot token lacks org admin) — that step is the
  owner's; the root-agent only *accepts* invites via the new bot's token.
- **A fine-grained PAT inherits nothing.** Granting the bot *account* access does not move its
  fine-grained *token*, and neither does an org base permission of `read`. Worse, an empty-selection
  fine-grained token still authenticates and still lists *public* repos, so it looks like it works.
  And one fine-grained token applies a single permission set to every repo it selects, so "brain +
  shared in one token" means **write on the shared governance repo**. Full detail and the definitive
  test are in `policies/github-access-policy.md` §Token type.
- **An empty brain repo provisions "successfully" into a broken agent** — dangling `~/CLAUDE.md`, no
  topics. Seed the brain first; see step (a2).
- **`loginctl enable-linger` is asynchronous.** It returns before the per-user systemd manager and its
  D-Bus socket exist, so a first provisioning can race it: `Failed to connect to bus` and no topics
  enabled. Current `provision-agent` starts `user@<uid>.service` and polls for
  `/run/user/<uid>/bus` before touching topics — if you are running an older copy that prints
  `[ok] topic <key> ()` with an **empty** state, that is this bug, and the topic is not running.

## Validation

- `claude-topic list` (as the agent) shows every topic `active` **with a non-empty sessionId** — an
  empty sessionId column means the session never registered, usually the unauthenticated-first-start
  case above.
- `git -C <a-repo> push --dry-run` works as the bot.
- The agent opens in the Claude web / mobile app.
- Re-running `ORG=<ORG> provision-agent <AGENT> <BRAIN>` reports only `[skip]`/`[ok]`, exits 0, and does
  not disturb the running session.
- `sudo agentic-divergence-check` is clean — it verifies `~/CLAUDE.md` resolves and that every file the
  bootstrap names exists, which catches a half-seeded brain.
