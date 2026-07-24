<!-- title: Runtime — sessions, supervision, and multi-device access -->
# Runtime

How an agent actually *runs* and stays reachable: persistent Claude Code sessions ("topics"),
supervised by systemd, driven from any device via Remote Control.

## Topics = sessions

A **topic** is one long-lived `claude --remote-control "<Display Name>"` process in an agent's home
directory. Each topic appears as its own entry in the Claude web/mobile session lists and is reachable
from every device at once — the same "one channel per subject" pattern you'd use in a chat app.

An agent can have several topics (e.g. a general one and a task-specific one). They're listed in the
brain's `deploy/topics.tsv` — a versioned `key<TAB>Display Name` registry.

## Supervision = systemd user units (no tmux)

Because Remote Control gives device access, there's no need for tmux. Each topic is a **systemd user
service** `claude-topic@<key>` that supervises the process directly:

- `Restart=always` → survives crashes.
- `loginctl enable-linger <user>` → the user's systemd runs at boot without an interactive login →
  survives reboot.
- The unit + registry are versioned in Git → portable.

Two details are baked into the unit because they bite otherwise:

- **pty:** Claude Code is a TUI — without a terminal it drops into non-interactive `--print` mode and
  exits. The unit runs it inside a pty via `/usr/bin/script -qfec "…" /dev/null`, staying foreground so
  systemd supervises Claude directly.
- **PATH + trust:** systemd's default PATH lacks `~/.local/bin` and `~/.local/node/bin`; the unit sets
  them. A headless Claude also blocks on the first-run "trust this folder?" prompt — pre-accept it once
  per agent home in `~/.claude.json`.

## The control surface = `claude-topic`

A single root-owned wrapper at `/usr/local/bin/claude-topic` drives the units. It's **root-owned** so
no agent can rewrite what another agent executes, and it's installed only by the explicit
[provisioning step](config-model.md) — never by the auto-sync.

```
claude-topic list                 # all topics: key, service state, sessionId, display name
claude-topic new <key> "<Name>"   # register + start under systemd, capture the new sessionId
claude-topic restart <key>        # systemctl restart — resumes the SAME conversation
claude-topic restart --new <key>  # explicit opt-in: start a FRESH conversation
claude-topic stop <key>           # stop + disable the service (sessionId is kept)
claude-topic remove <key>         # unregister for good (registry + state + unit)
claude-topic status <key>         # service state + stored/live sessionId
```

The wrapper's whole job is to **never lose conversation history**. It maps `key -> sessionId` in
`~/.config/agent/topics.state`, so a restart does `claude --resume <id> --remote-control '<Name>'` and
comes back as the same conversation. It fails fast rather than letting systemd fail-loop a topic:
unknown keys are rejected before `systemctl` is called, and a restart is refused when there's nothing
to resume (no stored session ID, or one whose transcript is missing) — both point you at `--new`, the
only way to deliberately start blank. These guards encode real incidents; keep them.

## Multi-device access = Remote Control

Every topic is reachable from claude.ai, the iOS/Android apps, and the desktop app — simultaneously.
This is the reason the happy path is Claude Code: you drive the same persistent agent from your phone
on the go and your laptop at the desk, with no extra gateway service to run, secure, and patch.

> One consequence to internalize: **anyone with the owner's Claude session can drive these agents** —
> and the privileged one can root the box by chat. That's why the privileged agent follows strict
> human-confirm gates and stays off broad content-ingesting integrations. See
> [`multi-agent-governance.md`](multi-agent-governance.md).

## Bring-up recap

`provision-agent` reads the brain's `deploy/topics.tsv` and `enable --now`s each topic, so a freshly
provisioned agent comes up with its sessions live. Verify with `claude-topic list`; each key should be
`active` with a session ID and open in the Claude apps.
