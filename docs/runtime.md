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
claude-topic urls                 # every topic's Remote Control URL (test them by hand)
claude-topic new <key> "<Name>"   # register + start under systemd, capture the new sessionId
claude-topic restart <key>        # systemctl restart — resumes the SAME conversation
claude-topic restart --new <key>  # explicit opt-in: start a FRESH conversation
claude-topic rotate <key>         # abandon the conversation, mint a NEW bridge id
claude-topic rotate-all           # rotate every enabled topic (what the boot unit runs)
claude-topic stop <key>           # stop + disable the service (sessionId is kept)
claude-topic remove <key>         # unregister for good (registry + state + unit)
claude-topic status <key>         # service state + stored/live/derived sessionId + bridge URL
claude-topic status <key> --porcelain   # the same, as field<TAB>value lines, for machines
claude-topic remember <key>       # persist the running topic's live sessionId to state
```

The wrapper maps `key -> sessionId` in `~/.config/agent/topics.state`, so a restart does
`claude --resume <id> --remote-control '<Name>'` and comes back as the same conversation. It fails
fast rather than letting systemd fail-loop a topic: unknown keys are rejected before `systemctl` is
called, and a restart is refused when there's nothing to resume (no stored session ID, or one whose
transcript is missing) — both point you at `--new`, the only way to deliberately start blank. These
guards encode real incidents; keep them.

### The pointer goes stale on its own, and that is the hard part

The runtime — not the wrapper — mints sessionIds, and **re-keys them silently on `/clear` and on
compaction**. If `topics.state` is written only when a wrapper verb runs, then from the moment the
runtime forks, the stored pointer is stale and nothing says so. The next restart resumes the stale
branch, orphans everything said since, and **reports success**, because by its own definition it
succeeded. In the reference deployment this happened six times across two agents before anyone
noticed; the largest instance lost 8.5 MB and three days, and one was caused by a self-restart
armed specifically to demonstrate that restarting was safe.

Two mechanisms close it, and they are deliberately independent:

- **A `SessionStart` hook** (`claude-topic-session-hook`, root-owned, registered in each brain's
  `deploy/claude-settings.json`) writes the state at the instant the runtime creates an id.
  `SessionStart` fires for `startup|resume|clear|compact` — exactly the set that mints ids.
- **`claude-topic run`** — the unattended path systemd re-enters on every restart — derives the
  live branch from disk and adopts it when the stored pointer is stale.

**Both identify the topic the same way: by the Remote Control bridge, never by configuration.**
The bridge is the durable identity a sessionId is not — the runtime re-keys the session on every
`/clear` but re-keys the bridge only when you deliberately start fresh — so *same bridge = same
topic*, and the newest transcript on that bridge is the live branch. Deriving from disk also means
it answers after a crash and at boot, when there is no live process to ask.

The hook's first version took the topic key from a systemd `Environment=TOPIC_KEY=%i` instead.
**Do not do this.** It was wrong in both directions: absent when needed (the units were running
from before the unit file changed, so twelve of twelve topics silently refused), and — worse —
present when not wanted, because a systemd `Environment=` is inherited by the entire process tree,
so any nested `claude` fires `SessionStart` carrying the topic's key and a *foreign* session id.
A binding that can be absent when you need it and present when you don't is not a binding.

The hook runs **detached**: `SessionStart` fires before the runtime has written the transcript, so
a synchronous hook loses the race by being early, and a wait long enough to win it would be paid
by every ordinary `claude` run that is not a topic at all.

**Order transcripts by mtime, not by the timestamp a transcript declares about itself.** That tree
is writable by the agent whose resume you are choosing, so a self-declared field is an instruction
from the data: a two-line file claiming a timestamp in 2099 wins outright. mtime defeats
pre-planting and nothing more — a file planted just before a restart gets mtime "now" for free.
That hole cannot be closed at this layer, so the control is **detection**: every adoption is
appended to `topics.rotated`, and the divergence check reports recent ones.

Finally: **watch the mechanism itself.** The hook was inert on every live topic for an afternoon
while every monitoring layer reported clean, because nothing asserted it was installed *and*
registered. Both halves fail silently on their own.

## `active` is not `reachable`

Preserving history was the wrapper's original whole job. One incident forced a second, competing
goal on it.

**The Remote Control bridge dies with the conversation, and nothing local can see it.** A topic can
be `active`, `enabled`, running under `Restart=always`, with a valid stored sessionId and a healthy
process — and its URL still answers *"session can't be found"* in the apps. The bridge id is
server-side state; `--resume` re-announces the id the session had, it does not re-establish a bridge
the server has already dropped. On 2026-08-06 the reference deployment came back from a reboot with
**11 of 12 topics `active` and unreachable**, and every check on the box was green.

So the honest statement of the limitation:

- `claude-topic urls` and `status` print what the session **announced**, not what the server still
  serves. There is no local probe that can tell the difference. **Opening the URL is the only proof.**
- `restart` is the wrong reflex here: it resumes the same conversation and therefore re-announces the
  same dead bridge. Only **`rotate`** abandons the conversation and mints a new bridge id. The
  abandoned sessionId is appended to `topics.rotated` first, so the history stays addressable.

Because the failure is invisible locally, the fix is not a check — it's an unconditional action:
**`claude-topic-rotate-on-boot.service` runs `rotate-all` once per boot**, so reachability after a
reboot never depends on a check that cannot see the failure. It writes a digest of what it abandoned
and the new URLs to `~/.config/agent/last-boot-rotation.tsv`.

The trade is deliberate and worth stating plainly: **every topic loses its conversation context on
every reboot.** Reboots are rare; a silently unreachable fleet is worse than a fresh one. Anything
that must survive a reboot belongs in the brain repo or in memory, not in a conversation — which is
the same rule the [memory model](memory.md) already asks you to follow.

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
provisioned agent comes up with its sessions live. It also `enable`s (not `--now`) the
rotate-on-boot unit, so the next reboot mints fresh bridges by itself.

Verify with `claude-topic list`; each key should be `active` with a session ID. Then verify the part
the box cannot verify for you: run `claude-topic urls` and **open one** in the apps. `active` is not
`reachable` — see above.
