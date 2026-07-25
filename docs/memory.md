<!-- title: Memory — two tiers, nightly "dreaming", human-gated audit -->
# Memory & Dreaming

Durable memory is a **two-tier** model, and improvement over time comes from a nightly consolidation
("dreaming") that is deliberately allowed to touch **only memory** — never identity or capabilities.

## Two tiers

- **Working tier — runtime auto-memory.** The agent runtime's own memory store (e.g. Claude Code's
  `~/.claude/.../memory`) is fast and auto-captured during sessions, but it is **not canonical** and
  does **not** travel with the brain across a re-provision or a framework switch.
- **Canonical tier — the Git brain.** `memory/distilled-memory.md` (compact, high-signal) +
  `memory/episodic/` (dated milestones) + `memory/auto/` (a machine-mirror of the working tier) —
  reviewed, portable, and inspectable/editable in GitHub. When the two disagree, Git wins.

High-signal facts are **distilled from the working tier into the canonical tier**, so the durable brain
— not a runtime cache — is the source of truth.

## Nightly "dreaming" (automatic, low-risk only)

> **⚠️ Experimental — not shipped in the templates.** The nightly job below exists in the reference
> implementation and is **deliberately not published here yet**: it is Claude-Code-specific (it depends
> on `claude -p` and three of its flags), it runs unattended as root, and it has only days of production
> evidence. **You do not need it to get the value of this framework** — the two-tier model above works
> with a periodic human-run pass (see the `agent-audit` runbook). It is described for reference so you
> can judge the design, not as a step to install.

A per-agent timer (`dream@<user>.timer`, ~05:00, opt-in, disabled by default) runs a `dream` script:

1. **Mirror (deterministic).** Copy the runtime auto-memory into `memory/auto/` and commit. No model, no
   judgment, no risk.
2. **Distil (scoped model run).** A tightly-scoped headless run promotes durable, recurring facts into
   `memory/distilled-memory.md` (with provenance), archives stale ones, and appends *suggestions*
   (candidate skills, identity nits) to `memory/dream-log.md` for later human review.

**Enforcement lives in the wrapper, not the prompt:** the script commits only `memory/` and reverts any
change the run makes outside it (preserving pre-run runtime churn). So dreaming can refine memory but can
**never** alter `SOUL.md`, `OPERATING.md`, skills, or config. Commits are prefixed `[dream]` — greppable
and mass-revertible. This mirrors the safe core of OpenClaw's "dreaming" (stage → gate → promote;
narrative kept out of the promotion path; previewable and reversible) and Letta-style consolidation as a
separate, asynchronous role that cannot touch the interactive identity.

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
  it** — an erosion tripwire, a liveness check and a weekly digest.
- It depended on vendor-specific CLI flags whose semantics could widen without erroring.
- The requirement it served — improvement with few interactions — turned out to be served about as well
  by a deterministic mirror plus a periodic human pass, at a fraction of the moving parts.

When a mechanism needs three watchers to be safe, the cheap fix is usually not to run it unattended. The
deterministic half — mirroring the runtime's working memory into Git — is kept and shipped as optional.
The model-driven half is documented here and deliberately left out of the templates.
