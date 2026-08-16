<!-- title: Portability — one canonical identity, many framework adapters -->
# Portability

A core reason this system exists: **your agents' accumulated knowledge should never be trapped in one
vendor's store.** Portability here has two independent axes — the *brain content* is portable across
frameworks, and the *deployment* is portable across machines. The runtime wiring is not portable, and
this page is explicit about where that line falls.

## Axis 1 — brain portability (across agent frameworks)

**`CLAUDE.md` is canonical, complete, and the only always-on file.** It holds *who the agent is*
(identity, voice, principles) and *what it does* (scope, delegation, threat model, human-confirm gates)
in plain, framework-agnostic Markdown. It is written with **absolute** `~/github/...` paths so it
resolves from any working directory; `provision-agent` symlinks it to `~/CLAUDE.md`, and the supervised
topic session runs with `WorkingDirectory=%h`, so that symlink is what the running agent loads. Editing
the repo edits the live contract — a symlink, never a fork. Shared facts about the owner and org live
once in `shared/owner-profile.md`.

```
  kb-agent-<role>-<name>/
    CLAUDE.md      ← the whole always-on contract — identity + scope + gates (the value)
    memory/        ← distilled-memory.md + auto/ (nightly machine mirror)
    skills/        ← folder-per-skill; the one retrievable layer the runtime advertises itself
    shared/        → sibling clone of the governance layer
```

> **Why one file and not two.** Through 0.6.x this framework split identity by *durability* —
> `SOUL.md` for who the agent is, `OPERATING.md` for what it does, `CLAUDE.md` a thin pointer at both.
> Measurement killed it (0.7.0): nothing but `CLAUDE.md` is auto-loaded, so the other two entered
> context only when the model chose to obey an instruction to read them — 16–54% of substantial
> sessions across the reference deployment — while 45–70% of each `OPERATING.md` was gates, threat
> model and delegation map. **A layer that loads only by instruction is not a layer, it is a
> suggestion.** Git already distinguishes what changes rarely from what changes often, for free.

> That was the second time this framework learned the same lesson. v0.6.0 had already collapsed a
> three-file *bootstrap* arrangement (`deploy/home-CLAUDE.md` plus repo-root `CLAUDE.md`/`AGENTS.md`
> adapters) for exactly the same reason: only one file was ever loaded, the others restated it, nothing
> caught them drifting — and they drifted. 0.7.0 finished the job on the *identity* layer.
>
> **And why Claude Code.** The *wiring* here is deliberately built around it, because **Remote Control**
> (sessions reachable from phone, web and desktop) is the reason the whole stack works the way it does;
> no other runtime currently offers an equivalent. What stays portable is the part with accumulated
> value: the brain content. Adopting another runtime means writing its entry file and rewriting the
> wiring — not migrating your memory, skills or identity.

### Why this matters beyond convenience
- **No lock-in.** Your agents' accumulated knowledge isn't trapped in one vendor's memory store.
- **Reviewable + versioned.** Identity changes are Git diffs, not opaque settings.
- **Editable from anywhere.** It's Markdown in GitHub — edit from a laptop or a phone.

## What is Claude-Code-specific (an honest list)

Framework-agnosticism here is a property of the **brain content**, not of the whole system. Being
straight about the split is what makes the claim usable:

**Moves unchanged to another framework** — `CLAUDE.md`, `memory/`, `skills/` bodies,
`tools/`, and everything in the shared governance layer. This is plain Markdown, and it is
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
| **MCP server configuration** | Lives in the runtime's user-scoped config, which also holds credentials — a `.mcp.json` in the brain repo is decorative unless you pass it explicitly with `--mcp-config`. See [`context-budget.md`](context-budget.md). | The box |
| `~/.config/agent/topics.state` (session IDs) | Per-machine runtime state | The box |
| `~/.config/agent/topics.rotated` | Per-machine log of abandoned session IDs | The box |
| `~/.config/agent/last-boot-rotation.tsv` | Per-boot digest written by rotate-on-boot | The box |
| `~/.claude/settings.local.json` | Auto-accumulated per-session approvals | The box, git-ignored |

Everything else is a `git clone` away.

## The takeaway

Portability isn't a feature bolted on; it *is* the architecture — but it is specific about what it
covers. Canonical identity in framework-agnostic Markdown + thin adapters means the *brains* would
follow you to another runtime (you would rewrite the wiring, not the agents); Git-as-source-of-truth +
explicit provisioning gives you real machine freedom, exercised. The happy path is Claude Code because
of Remote Control ([`runtime.md`](runtime.md)).
