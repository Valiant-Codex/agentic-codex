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
output is a working bot with a token to be restored on-box (step (c) fails fast without it). Also do the
first-ever interactive Claude login for the user (owner's account, once) so the runtime is authorized.

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

From `templates/infra/scripts/`, run:

```bash
ORG=<ORG> provision-agent <AGENT> <BRAIN>
```

It is idempotent (safe to re-run) and does everything git carries: clones the brain + `kb-agent-shared`
sibling, wires the symlinks (`shared`, `~/CLAUDE.md`, `~/.local/bin/claude-topic`), installs the
root-owned `claude-topic` wrapper + per-user systemd unit, materializes `~/.claude/settings.json` as a
real copy, enables linger, and starts each topic in `deploy/topics.tsv`. It fails fast if the bot token
is missing (that is why (a)/(b) come first). **Do not re-do the clone/symlink/unit/topic steps
manually** — reference `provision-agent`.

The agent is then auto-kept-current by `kb-sync`. Brain **content** (SOUL.md/OPERATING.md identity,
tools, skills) is authored by the agent (or its designated maintainer), not by the root-agent.

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
# 2. Back up the home if anything un-pushed matters, then remove the user
sudo tar czf /root/backup-<AGENT>-$(date +%F).tgz /home/<AGENT>   # keep until sure
sudo userdel -r <AGENT>
```
3. GitHub (owner, or a token with org admin): **revoke the bot PAT**, remove the bot as collaborator or
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

## Validation

`claude-topic list` shows every topic `active` with a sessionId; `git -C <a-repo> push --dry-run` works
as the bot; the agent opens in the Claude web / mobile app.
