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
  `memory/episodic/` (dated milestones) + `memory/auto/` (a machine-mirror of the working tier) —
  reviewed, portable, and inspectable/editable in GitHub. When the two disagree, Git wins.

High-signal facts are **distilled from the working tier into the canonical tier**, so the durable brain
— not a runtime cache — is the source of truth.

## The nightly mirror (optional, deterministic)

> **Optional.** Ships with the timer **disabled**; enable per agent when you want it. It is Claude-Code
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
