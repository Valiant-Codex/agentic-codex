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
- **Adapters are thin and per-framework.** Each runtime reads a different entry file; each is a short
  pointer to `SOUL.md` + `OPERATING.md`:
  - `CLAUDE.md` → the adapter Claude Code reads first.
  - `AGENTS.md` → the adapter several other tools (Codex, Cursor, and others) read.
  - add another adapter file for another framework — the adapter itself is ~15 lines. The *runtime
    wiring* underneath it is not: see the honest accounting below.

```
      SOUL.md + OPERATING.md          ← canonical identity + contract (the value)
          ▲        ▲        ▲
     CLAUDE.md  AGENTS.md  <other>.md   ← thin adapters, one per framework
   (Claude Code) (Codex/…)  (future)
```

Switching or adding a framework means writing one small adapter **and rewriting the runtime wiring** —
but not migrating your memory, skills or identity. Your
`memory/`, `skills/`, `tools/`, and `context/` are already just Markdown — nothing framework-specific
to port.

On the box, the running location is a **symlink or installed copy**, never a fork of the content:
`~/CLAUDE.md` points at `deploy/home-CLAUDE.md` in the brain repo, so editing the repo edits the live
bootstrap.

### The three bootstrap files, and which one actually runs

A brain carries up to three entry points, and it is worth being precise about who reads each — they are
easy to mistake for duplication:

| File | Read when | Read by |
|---|---|---|
| `deploy/home-CLAUDE.md` | the supervised topic session runs (`WorkingDirectory=%h`, so cwd is `~`) | **always, in normal operation** |
| `CLAUDE.md` (brain repo root) | someone runs Claude Code with cwd *inside* the brain repo | a human working on the brain |
| `AGENTS.md` (brain repo root) | the same, under a non-Claude runtime (Codex, Cursor, …) | that runtime |

Because the topic unit sets `WorkingDirectory=%h`, **only `deploy/home-CLAUDE.md` is loaded by the
running agent.** The two repo-root files are *adapters*: they exist so the brain stays usable from
inside the repo and portable across runtimes.

> ⚠️ **If you only ever use one runtime and never open Claude Code inside the brain repo, the repo-root
> adapters are dead weight — and they will drift.** They restate what `home-CLAUDE.md` says, nothing
> loads them, so nothing catches them going stale. A specific trap: the root adapter naturally uses
> repo-relative paths (`SOUL.md`, `shared/bootstrap.md`), which do **not** resolve from `~` — so if you
> later "consolidate" by pointing `~/CLAUDE.md` at the root `CLAUDE.md`, the references break silently.
>
> Choose deliberately: either keep the adapters and accept they need syncing (worth it for genuine
> multi-runtime portability, which is the point of `AGENTS.md`), or drop them and keep
> `deploy/home-CLAUDE.md` as the single bootstrap. Do not keep them by accident.

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

**Status of `AGENTS.md`:** shipped as a thin adapter for frameworks that read it (Codex, Cursor and
similar), and kept deliberately short so it cannot become a second source of truth. It has **not been
exercised in production** — only Claude Code has. Treat it as a head start, not a guarantee.

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
