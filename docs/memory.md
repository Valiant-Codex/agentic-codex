<!-- title: Memory — two tiers, a deterministic mirror, and a human-gated audit -->
# Memory & Dreaming

Durable memory is a **two-tier** model: the runtime's own memory is a fast working cache, and Git is
canonical. A nightly job makes the working tier durable; a periodic human pass curates it. Nothing
scheduled ever touches identity or capabilities.

## Two tiers

- **Working tier — runtime auto-memory.** The agent runtime's own memory store (e.g. Claude Code's
  `~/.claude/.../memory`) is fast and auto-captured during sessions, but it is **not canonical** and
  does **not** travel with the brain across a re-provision or a framework switch.
- **Canonical tier — the Git brain.** `memory/distilled-memory.md` (compact, high-signal) +
  `memory/auto/` (a machine-mirror of the working tier) —
  reviewed, portable, and inspectable/editable in GitHub. When the two disagree, Git wins.

High-signal facts are **distilled from the working tier into the canonical tier**, so the durable brain
— not a runtime cache — is the source of truth.

## The one hard limit, and what manages growth

Everything above is about *what* is stored. This is the constraint on *how much*, and it is the
runtime's, not yours: Claude Code loads only the first **200 lines or 25 KB of `MEMORY.md`** at
session start and silently drops the rest. A memory past that point still exists on disk and is still
mirrored into Git — but nothing puts it in front of a fresh session. That is the one always-on number
with a real loss attached, so it is the one the [divergence check](divergence-check.md) asserts.

Two things follow, and both are easy to get backwards.

**Do not set a total context budget.** The obvious move is to add up `CLAUDE.md`, its imports and the
index and assert a ceiling. The reference deployment did exactly that, moved the number three times
in three days as it stung (20 → 30 → 40 KB), and retired it: it derived from nothing, it fired long
before the real cliff, and its remedy — shorten the index — was the one action that loses memories.
Worse, a budget is read as a target: that fleet over-trimmed its curated memory file twice, both
times by a session dutifully making a number go down. Report the total as a trend; assert the cliff.

**Growth is a human pass, not a threshold.** The [`agent-audit`](skills.md) memory pass retires dead
episodic index lines — a handoff that landed, a WIP whose work shipped — by deleting the *index line*
while the memory file stays, and reduces curated bullets to verdict + pointer wherever the reasoning
already sits in a named decision record. A threshold cannot tell a dead handoff from a standing rule;
a person reading the diff can. And it must never be the session that happened to notice the number
doing the trimming — that session is the one with a reason to hurry.

## The nightly mirror (optional, deterministic)

> **Enabled by `provision-agent`, with disclosure.** Provisioning an agent turns its mirror timer on
> and prints exactly what the job writes and how to switch it off (`sudo systemctl disable --now
> memory-mirror@<user>.timer`) — it is an unattended writer, and leaving it opt-in is how an agent
> once ran for a day with durability silently off. It is Claude-Code
> specific only in the path it reads (that runtime's memory store).

A per-agent timer (`memory-mirror@<user>.timer`, ~05:00, staggered) runs `memory-mirror`, which copies
the runtime's auto-memory into the brain repo at `memory/auto/`, then commits and pushes. That is all:
`rsync` + `git`. No model, no judgement, no nondeterminism — which is exactly why it is safe to leave
running unattended.

Two guarantees make it trustworthy:

- **Secret scan before staging.** Session-derived content can contain credentials, and a secret in Git
  is not undoable. A hit aborts before any commit and reports it for rotation.
- **Fail closed.** Every error path exits non-zero, so the unit lands in `failed` and your monitor
  surfaces it. Silence must never be mistaken for success.

Commits are prefixed `[mirror]`, so the machine's writes are greppable and mass-revertible.

**Why it is named for what it does.** This job was originally called `dream`, and it *did* once include
a model-driven "reflect and consolidate" pass. When that pass was removed (below), the name kept
promising a magic that was no longer there — so it was renamed. If reflection ever comes back, it comes
back as its own script with its own honest name.

## The autonomy line

**Memory = automatic; capability/identity = never unattended.** Anything scheduled is path-scoped to
`memory/` by its wrapper, so the strongest thing it can do about a skill or an identity file is leave a
suggestion. Those change only in a session with the owner present — ad hoc, or as the periodic
[`agent-audit`](skills.md) sweep — always as reviewable commits.

### A negative result worth publishing

Fully-unattended consolidation **was built, run, evaluated, and not adopted.** The accounting, because
it is more useful to you than the code would have been:

- Its measurable value rested on a *single* good run — it did produce one genuinely useful observation.
- Its cost was concrete: three defects found within hours of writing it (a destructive worktree clean
  that deleted untracked work, a `git` capability the model did not need, and every error path exiting
  zero so failure looked like success), and then **three further mechanisms invented only to supervise
  it** — an erosion tripwire and a liveness check, surfaced by the daily divergence report.
- It depended on vendor-specific CLI flags whose semantics could widen without erroring.
- The requirement it served — improvement with few interactions — turned out to be served about as well
  by a deterministic mirror plus a periodic human pass, at a fraction of the moving parts.

When a mechanism needs three watchers to be safe, the cheap fix is usually not to run it unattended. The
deterministic half — mirroring the runtime's working memory into Git — is kept and shipped as optional.
The model-driven half is documented here and deliberately left out of the templates.
