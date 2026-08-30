---
name: topic-management
description: Manage your own Remote Control topic sessions with the root-owned `claude-topic` wrapper — list, create, restart, stop, rename, remove, and recover a topic the app says it cannot find. Self-only, each agent manages its own topics; cross-agent work goes through the privileged agent. Use when the owner asks in session to add / rename / remove / restart / list one of your topics, or when a topic is active but unreachable.
type: skill
title: Topic Management
tags:
- skill
- topics
- remote-control
- systemd
- sessions
status: active
timestamp: 2026-08-16T00:00:00Z
---
# Topic Management

Your topic sessions — the conversations the owner opens from the web and mobile apps — are systemd
**user** services (`claude-topic@<key>`), each running `claude --remote-control "<Display Name>"` in a
pty. The control surface is `/usr/local/bin/claude-topic`: **root-owned**, run as yourself, no sudo.

It resolves *your own* brain repo from your Unix username (`kb-agent-*-$(id -un)`) and reads
`deploy/topics.tsv` there. That is the whole scoping mechanism — you cannot address another agent's
topics with it even by accident.

## Scope: your own topics only

**Each agent manages its own. Cross-agent is the privileged agent's.** If the owner asks you to touch
another agent's topic, say so and hand it over — do not try to reach it. The privileged agent drives
the same wrapper as that agent's Unix user (`manage-agents`).

## Commands

```bash
claude-topic list                       # key, state, sessionId, display name
claude-topic urls                       # every topic's Remote Control URL
claude-topic status <key>               # service state + stored/live/derived sessionId
claude-topic status <key> --porcelain   # the same as field<TAB>value lines, with a drift verdict
claude-topic new <key> "<Display Name>" # register + start, and commit topics.tsv
claude-topic restart <key>              # resumes the SAME conversation
claude-topic restart --new <key>        # start a FRESH conversation (see below)
claude-topic stop <key>                 # stop + disable; state survives, resumable later
claude-topic remove <key>               # ⚠️ unregister for good — registry + state + unit
claude-topic rotate <key>               # abandon the conversation, get a NEW bridge id
claude-topic remember <key>             # record the running topic's live sessionId
```

Keys allow letters, digits, `.`, `_`, `-`. Neither a key nor a display name may start with `-`; the
wrapper refuses, because `claude` would parse it as a flag rather than as the value.

## Before any restart / stop / rotate / remove: is it *you*?

**The wrapper has no self-guard.** Nothing stops you from stopping or removing the very topic your
session is running in — and the process executing the command is the one being killed. The command
dies mid-flight, and for `remove` it can die *after* disabling the unit and *before* committing.

So check first, always:

```bash
sed -n 's|.*/claude-topic@\([^/]*\)\.service.*|\1|p' /proc/self/cgroup
```

That prints the key you are living in. If it matches the target, **stop and hand it to the owner** —
it can be run from another topic, or by the privileged agent. Read-only commands (`list`, `urls`,
`status`) are always safe on yourself.

## Creating a topic

```bash
claude-topic new research "<Agent> (research)"
```

It registers the row, starts the unit, waits for the sessionId, and **commits `deploy/topics.tsv`
itself**. Do not stage or commit that file by hand. On any failure it rolls back both sides — registry
row and unit — so a half-built topic never survives.

Two things to say out loud to the owner before creating one, because neither is obvious:

- a topic is a **permanently-on session**, reachable from the web and mobile apps, restarted on boot;
- it costs context and money continuously, not per use.

If the key already has a stored sessionId, `new` **resumes** that old conversation instead of starting
blank — the wrapper warns, but read the output.

## Renaming

There is no `rename` verb. Edit the display name in `deploy/topics.tsv` (a TAB between key and name),
commit it as yourself, then `claude-topic restart <key>`. The name is read at process start, so
nothing changes until the restart.

## Removing — human-confirm with the owner first

`remove` is destructive and is **not** something to do on your own initiative. Show `claude-topic
status <key>` first, then wait for the owner's explicit go-ahead in session.

It unregisters the row, clears the stored sessionId and disables the unit. The transcript stays on
disk, but nothing resumes it any more — practically, the conversation is gone. Prefer `stop` when the
intent is "not now": it disables the topic and keeps it resumable.

## When a topic is active but the app says the session cannot be found

That is a stale bridge id, not a dead process — `restart` will not fix it, because it resumes the same
conversation and the same bridge. Use `rotate <key>`: it abandons the conversation (appending the old
sessionId to `~/.config/agent/topics.rotated`, so it stays addressable) and comes back with a new
bridge id and a new URL. Being `active` and `enabled` does **not** mean reachable.

## Gotchas

- **`restart` refuses to start a blank conversation.** With no stored sessionId it captures the live
  one first, and rather than guess it errors out. `--new` is the deliberate override and **orphans the
  history**. Never reach for it just to clear an error.
- **`deploy/topics.tsv` must be committed**, and the wrapper does it for you. A topic living only in an
  uncommitted registry can be wiped by any `git checkout`, leaving the unit fail-looping on "no display
  name". If the wrapper warns that the commit was local-only, fix it before moving on.
- **If your repo already had unpushed commits**, the wrapper commits but deliberately does **not**
  push, so it doesn't publish unrelated half-finished work. Push it yourself when your tree is sane.
- **A dirty worktree makes the sync timer skip your repo silently.** Leaving `topics.tsv` uncommitted
  stops your brain from syncing at all, on a log line nobody reads.
- **Check `drift` before you restart anything.** `claude-topic status <key> --porcelain` prints a
  computed verdict: `no` means the stored pointer is the conversation you are actually having, `yes`
  means it is not and a restart would adopt the newer branch, `unknown` means it could not tell —
  which is **not** the same as clean. Do not read this out of the prose output with a regex; that is
  how a guardian ended up silently reporting healthy for a stopped topic.
- **MCP config changes need a restart** — MCP servers load at process start.
- **Never restart a topic to "apply" a change you have not committed.** The unit re-reads the repo; an
  uncommitted change is not there.

## MCP servers are a standing context tax — park what you do not use

Every MCP tool **name** sits in the always-on context at roughly 16 tokens, whether or not you touch
that server. One ~546-tool server measured 8,824 tokens — 23% of a whole session — paid on every
session forever. `scripts/mcp-park` moves a server's definition between `~/.claude.json` (attached)
and `~/.config/agent/mcp-parked.json` (parked), atomically and **without ever printing it**: the
definition normally carries an API key.

```bash
scripts/mcp-park status <server>                  # attached / parked / not configured
scripts/mcp-park on  <server> --restart <topic>   # attach, then restart so it loads
scripts/mcp-park off <server> --restart <topic>   # park it again when the work is done
```

The restart is what makes it take effect, which is why this lives here rather than in a skill of its
own. Afterwards, **open the topic's URL** — a restart cannot prove the bridge is still served.

## Validation

`claude-topic list` shows the topic `active` with a sessionId; `claude-topic urls` gives a URL that
actually opens; `git status` in your brain repo is clean afterwards.

## Related

- `manage-agents` (privileged agent only) — the full agent lifecycle, and cross-agent topic work.
- `deploy/topics.tsv` in your own repo — the versioned registry; `~/.config/agent/topics.state` holds
  the unversioned `key→sessionId` runtime state.
