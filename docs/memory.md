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

A per-agent timer (`dream@<user>.timer`, ~05:00, opt-in, disabled by default) runs `infra/scripts/dream`:

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

**Memory = automatic; capability/identity = human-gated.** The nightly job *proposes* (memory + a
dream-log); the [`agent-audit`](skills.md) skill — which the owner triggers — *disposes*, turning
suggestions into real skill/identity changes with the owner approving each diff. This gives
improvement-over-time with minimal interaction while structurally preventing self-rewriting drift, which
the memory-poisoning and agent-misevolution literature is unanimous must be gated and enforced *outside*
the agent.
