<!-- title: Divergence check — invariants, not vocabulary -->
# The divergence check

Your brains are Markdown, so nothing stops them from quietly disagreeing with reality: a renamed file
leaves dangling references, a skill stops being discoverable, an adapter names a file that no longer
exists, a clone stops syncing. None of that raises an error — the agent just gets a little less correct
every week. `agentic-divergence-check` is a **read-only** daily linter for exactly that class of rot.

It **reports, it never fixes.** Fixes go through a human (or the `agent-audit` skill), because deciding
*what* the truth should be is judgement, not automation.

## What it checks

Per agent brain:

- **Declared shape** — `SOUL.md`, `OPERATING.md`, `CLAUDE.md` exist, and the adapter actually loads
  SOUL + OPERATING + `shared/owner-profile.md`.
- **Skills are discoverable** — folder-per-skill with a `SKILL.md`, frontmatter `name` matching the
  folder, non-empty `description`, and no flat `skills/*.md` left behind.
- **Runtime registration** — `~/.claude/skills` *is* a symlink, resolves, and points at this brain's
  `skills/`. (Absent or dangling is exactly when skills are invisible, so both are failures.)
- **Every symlink resolves** — catches a fleet-common skill renamed in the shared layer.
- **The live entry point** — `~/CLAUDE.md` resolves to `deploy/home-CLAUDE.md`, and every file that
  bootstrap names exists.
- **Clone agrees with origin** — nothing unpushed (a silently failing push) and nothing behind (a repo
  the sync has been skipping).
- **References resolve** — every relative path referenced in active Markdown exists. Renames self-report.
- **Roster vs reality** — the declared fleet and the brains on disk agree, in both directions.

## The design rule: invariants, not vocabulary

An earlier version also grepped for stale *technology words* (an old VCS command, an old session
multiplexer, a removed filename). That was a maintenance trap, and it is worth stating why so you don't
rebuild it:

- The list encodes today's stack. When you change tools, nobody prunes it.
- It then passes because it no longer matches anything — **false confidence, worse than no check.**
- It can flag *true* content: an agent's own memory of a past migration legitimately mentions the old
  tool, and the checker would demand you censor an accurate memory.
- And "this document describes the old way" is a semantic judgement — that belongs to a human review
  pass, not a grep.

So everything it checks is a structural invariant that stays true regardless of which technologies you
use. It does not need pruning as you evolve.

## Wiring it

```bash
agentic-divergence-check          # run it by hand; exits 0 clean, 1 on drift
systemctl enable --now agentic-divergence-check.timer   # daily, once your fleet passes
```

Two deliberate choices in the unit: `SuccessExitStatus=1`, so **drift does not page you** (a stale doc
link must never wake anyone, and a failed oneshot would stay failed), and the weekly
`agentic-update-check` report carries the finding count **and alarms if it is non-zero** (or if the
check itself crashed), so drift surfaces on a human cadence instead of being discarded.
A real malfunction — any other non-zero exit — still fails loudly.

**Get it clean, then keep it clean.** A check that always reports something trains you to ignore it.
