<!-- title: Portability — one canonical identity, many framework adapters -->
# Portability

A core reason this system exists: **you should be able to change the agent runtime without rewriting
your agents.** Portability here has two independent axes — the *brain* is portable across frameworks,
and the *deployment* is portable across machines.

## Axis 1 — brain portability (across agent frameworks)

The trick is a strict split between the canonical identity and the runtime adapter.

- **`system-prompt.md` is canonical.** It holds the agent's identity, voice, mission/scope,
  human-confirm gates, and bootstrap order — as plain, framework-agnostic Markdown. This is the file
  that carries the value.
- **Adapters are thin and per-framework.** Each runtime reads a different entry file; each is a short
  pointer to `system-prompt.md`:
  - `CLAUDE.md` → the adapter Claude Code reads first.
  - `AGENTS.md` → the adapter several other tools (Codex, Cursor, and others) read.
  - add another adapter file for another framework — it's ~15 lines.

```
          system-prompt.md          ← canonical identity (the value)
          ▲        ▲        ▲
     CLAUDE.md  AGENTS.md  <other>.md   ← thin adapters, one per framework
   (Claude Code) (Codex/…)  (future)
```

Switching or adding a framework means writing one small adapter, not migrating your memory. Your
`memory/`, `skills/`, `tools/`, and `context/` are already just Markdown — nothing framework-specific
to port.

On the box, the running location is a **symlink or installed copy**, never a fork of the content:
`~/CLAUDE.md` points at `deploy/home-CLAUDE.md` in the brain repo, so editing the repo edits the live
bootstrap.

### Why this matters beyond convenience
- **No lock-in.** Your agents' accumulated knowledge isn't trapped in one vendor's memory store.
- **Reviewable + versioned.** Identity changes are Git diffs, not opaque settings.
- **Editable from anywhere.** It's Markdown in GitHub — edit from a laptop or a phone.

## Axis 2 — deployment portability (across machines)

The [config model](config-model.md) guarantees that everything except secrets and volatile runtime
state lives in Git. So moving to a new VPS (or recovering a dead one) is:

1. Create the Unix user(s); install Claude Code (+ Node if the agent uses `npx`-based MCP servers).
2. **Restore the agent's secret** (its `gh` token; MCP secrets) — out-of-band, from your secret store
   (see [`secrets.md`](secrets.md)).
3. `ORG=<ORG> provision-agent <user> <brain>` — clones brain + `kb-agent-shared`, wires the symlinks,
   installs the root-owned wrapper + systemd unit + settings, enables linger, starts the sessions.

`provision-agent` is idempotent, so the same command also *converges* an existing box back to the Git
state. Topic session IDs are per-machine runtime state (not carried) — a brand-new box simply starts
fresh conversations.

## What is deliberately **not** portable

| Not carried | Why | Where it lives |
|---|---|---|
| Secrets (tokens, keys) | Never in Git | Your secret store ([`secrets.md`](secrets.md)) |
| `~/.claude.json` account/session state | Per-machine, account-bound | The box |
| `~/.config/agent/topics.state` (session IDs) | Per-machine runtime state | The box |
| `~/.claude/settings.local.json` | Auto-accumulated per-session approvals | The box, git-ignored |

Everything else is a `git clone` away.

## The takeaway

Portability isn't a feature bolted on; it *is* the architecture. Canonical identity in
framework-agnostic Markdown + thin adapters gives you runtime freedom; Git-as-source-of-truth +
explicit provisioning gives you machine freedom. The happy path is Claude Code because of Remote
Control ([`runtime.md`](runtime.md)) — but the brains would follow you elsewhere.
