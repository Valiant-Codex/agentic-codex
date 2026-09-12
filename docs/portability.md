<!-- title: Portability — machines yes, harnesses no; the ideas travel -->
# Portability

Two claims used to live on this page. One is still true and exercised: **the deployment moves between
machines** — everything but secrets and volatile runtime state is in Git, and one command rebuilds a box.
The other — that the *brain content* moves unchanged to another agent framework — is retired as of
0.10.1, because it stopped being true as the framework grew. This page is explicit about both.

## Axis 1 — what does and does not port across harnesses

**This framework runs on Claude Code and nothing else.** That was always true of the wiring; it is now
true of the brain content too. What the brains contain is written for this harness:

| Piece of a brain | Why it is Claude Code's, not generic Markdown |
|---|---|
| `CLAUDE.md` | The one file the harness auto-loads, with its `@`-import syntax for the distilled memory and the runtime's own rules about what reaches context |
| `memory/auto/` | A nightly mirror of the harness's **auto-memory store**, whose index has the harness's hard read limit — the memory model is built around that cliff ([`memory.md`](memory.md), [`context-budget.md`](context-budget.md)) |
| `skills/*/SKILL.md` | Anthropic's Agent Skills format, discovered through `~/.claude/skills` |
| `decision-loop`, `advisor-review`, `agent-audit` | Written against the harness's sub-agent tool: an advisor pass *is* a fresh sub-agent |
| `deploy/claude-settings.json`, the SessionStart hook | The harness's settings schema; the hook that keeps `topics.state` honest fires on the harness's own session events |
| `deploy/topics.tsv` and everything a topic is | A topic is a `claude --resume … --remote-control` session; its identity is the Remote Control bridge ([`runtime.md`](runtime.md)) |

Point another harness at one of these repos and you get Markdown it cannot act on. Nothing here has been
exercised on another harness, and the framework no longer claims it could be.

**What travels are the ideas**, and they are the reason the repo exists — each is a pattern you can
rebuild on any harness that can read a file and run a process:

- **A brain is a Git repo of Markdown**, per agent, on GitHub: identity, memory and skills you read,
  diff, review and edit from a phone. Not a vendor's memory store.
- **A shared governance repo** (`kb-agent-shared`: owner profile, policies, decision records, fleet-common
  skills) reached from every brain, refreshed by a sync that only ever pulls inert data, written by exactly
  one agent.
- **One explicit provisioning step** applies Git to a live box; nothing on the box is canonical
  ([`config-model.md`](config-model.md)).
- **Persistent agents as supervised services**, one Unix user each, one privileged and the rest not
  ([`multi-agent-governance.md`](multi-agent-governance.md)).
- **A nightly memory mirror** from the harness's store into the brain repo, so memory survives the box.
- **A structural drift check** that asserts invariants, never vocabulary ([`divergence-check.md`](divergence-check.md)).
- **A dead-man's switch** where silence is the alarm ([`monitoring.md`](monitoring.md)).
- **Decision records and a human-gated autonomy line** as the way an owner and a fleet stay coherent.

That is the portable part: the shape. The files are Claude Code's.

## Why Claude Code

**Remote Control.** Sessions reachable from phone, web and desktop, with the process and the filesystem
staying on your machine, no gateway of your own to run and patch. No other harness offers an equivalent,
and it is the reason the whole stack has the shape it has. Two consequences worth knowing before you
adopt: remote attach is a *harness* property, not a model property — put a different model behind Claude
Code and Remote Control switches off; and the harness's resume semantics (which conversation a session
reconnects to after a restart or a reboot) changed four times in thirty minor versions, so this framework
states the runtime version it depends on ([`runtime.md`](runtime.md)).

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
| `~/.config/agent/last-boot-rotation.tsv` | Digest written by `rotate-all` when someone runs it (per-boot until 0.10.0) | The box |
| `~/.claude/settings.local.json` | Auto-accumulated per-session approvals | The box, git-ignored |

Everything else is a `git clone` away.

## The takeaway

Portability here is the machine axis, exercised: Git-as-source-of-truth plus one explicit provisioning
step means a dead VPS is a clone away. Across harnesses, what you keep is the shape — brains in Git, a
shared governance repo, one provisioning boundary, supervised agents, a mirror, a drift check, a
dead-man's switch — and you rewrite the files for whatever you run. The files here are Claude Code's, on
purpose, because of Remote Control ([`runtime.md`](runtime.md)).
