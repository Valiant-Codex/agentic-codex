<!-- title: Portability — one canonical identity, many framework adapters -->
# Portability

A core reason this system exists: **your agents' accumulated knowledge should never be trapped in one
vendor's store.** Portability here has two independent axes — the *brain content* is portable across
frameworks, and the *deployment* is portable across machines. The runtime wiring is not portable, and
this page is explicit about where that line falls.

## Axis 1 — brain portability (across agent frameworks)

The trick is a strict split between the canonical identity and the runtime adapter.

- **`SOUL.md` + `OPERATING.md` are canonical**, split by *durability*. `SOUL.md` holds *who the agent
  is* (identity, voice, principles) — durable, changed rarely; `OPERATING.md` holds *what it does*
  (mission/scope, threat model, human-confirm gates, bootstrap) — the operating contract. Both are
  plain, framework-agnostic Markdown. Splitting durable identity from mutable instructions keeps
  identity from drifting when operational details change. Shared facts about the owner/org live once in
  `shared/owner-profile.md`.
- **One thin bootstrap.** The brain's root `CLAUDE.md` is the **single runtime entry point**: a short
  pointer to `SOUL.md` + `OPERATING.md` + `shared/owner-profile.md`, written with **absolute**
  `~/github/...` paths so it resolves from any working directory. `provision-agent` symlinks it to
  `~/CLAUDE.md`; the supervised topic session runs with `WorkingDirectory=%h`, so that symlink is what
  the running agent loads. Editing the repo edits the live bootstrap — a symlink, never a fork.

```
      SOUL.md + OPERATING.md      ← canonical identity + contract (the value)
                ▲
            CLAUDE.md             ← one thin bootstrap (symlinked to ~/CLAUDE.md)
           (Claude Code)
```

> **Why only one file, and why Claude Code.** Earlier versions shipped a three-file arrangement
> (`deploy/home-CLAUDE.md` as the runtime bootstrap plus repo-root `CLAUDE.md`/`AGENTS.md`
> per-runtime adapters). It was dropped (v0.6.0): only one file was ever loaded in normal operation,
> the others restated it, nothing caught them drifting apart — and they drifted. This framework's
> *wiring* is deliberately built around Claude Code, because **Remote Control** (sessions reachable
> from phone/web/desktop) is the reason the whole stack works the way it does; no other runtime
> currently offers an equivalent. What stays portable is the part with accumulated value: the brain
> content below. Adopting another runtime means writing its entry file and rewriting the wiring —
> not migrating your memory, skills or identity.

### Why this matters beyond convenience
- **No lock-in.** Your agents' accumulated knowledge isn't trapped in one vendor's memory store.
- **Reviewable + versioned.** Identity changes are Git diffs, not opaque settings.
- **Editable from anywhere.** It's Markdown in GitHub — edit from a laptop or a phone.

## What is Claude-Code-specific (an honest list)

Framework-agnosticism here is a property of the **brain content**, not of the whole system. Being
straight about the split is what makes the claim usable:

**Moves unchanged to another framework** — `SOUL.md`, `OPERATING.md`, `memory/`, `skills/` bodies,
`tools/`, `context/`, and everything in the shared governance layer. This is plain Markdown, and it is
where the accumulated value lives.

**Would have to be rewritten** — the runtime wiring:

| Piece | Why it's runtime-specific |
|---|---|
| `claude-topic` (the session wrapper) | Captures and resumes Claude Code session IDs, drives `--remote-control`, allocates a pty |
| `claude-topic@.service` | Supervises that wrapper |
| `deploy/claude-settings.json` | Claude Code's settings/permissions schema |
| `~/.claude/skills` symlink | How Claude Code discovers skills |
| `SKILL.md` frontmatter | Anthropic's Agent Skills format |
| `~/.claude.json` trust/onboarding flags | Claude Code first-run state |
| Remote Control itself | The multi-device access path, and the reason this happy path is Claude Code |


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

Portability isn't a feature bolted on; it *is* the architecture — but it is specific about what it
covers. Canonical identity in framework-agnostic Markdown + thin adapters means the *brains* would
follow you to another runtime (you would rewrite the wiring, not the agents); Git-as-source-of-truth +
explicit provisioning gives you real machine freedom, exercised. The happy path is Claude Code because
of Remote Control ([`runtime.md`](runtime.md)).
