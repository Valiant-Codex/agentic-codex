# Changelog

All notable changes to **agentic-codex** are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/), and the project follows
[Semantic Versioning](https://semver.org/). Pre-1.0: the framework is still evolving, so minor
versions may include structural changes. `1.0.0` is reserved for a deliberate "stable and proven"
milestone.

## [0.4.1] — 2026-07-26 — Harden `claude-topic` against a writable-by-the-agent-it-manages threat model

A security/robustness release, ported verbatim (modulo the `<org>`/`github/*` placeholders) from the
reference deployment after two rounds of adversarial review — an independent agent prompted to refute
each fix, not confirm it — caught real problems in the reviewed code before they shipped. Every finding
below was reproduced by hand before being accepted; see `docs/` for the fuller incident writeups if this
project starts one.

The thing that makes `claude-topic`/`agentic-monitor` a different class of script from most of this
repo: `topics.tsv` and `topics.state` are **writable by the very agent whose sessions they control**, and
that agent is an LLM that reads untrusted web/repo/log content. A prompt injection with write access to
the agent's own home directory is a realistic threat here, not a hypothetical one.

### Fixed
- **`save_sid` could silently collapse `topics.state` to one row.** An unreadable/corrupted state file
  masked its own read failure with `|| true`, so the next write replaced the *entire* file with just the
  one row being saved — every other topic's sessionId gone, each one's next restart starting a BLANK
  conversation (the exact failure this project's incident-2026-07-17 guard exists to prevent, just
  reached a different way). `topics.tsv`'s append had the same shape of bug: appending onto a file
  missing its trailing newline corrupted the previous row instead of adding a new one. All writes are now
  lock-serialized (`flock` + a private `mktemp`, not a shared fixed tmp name — two concurrent invocations
  could otherwise steal each other's tmp file mid-write) and abort on a read failure instead of masking
  one.
- **A display name or sessionId starting with `-` was parsed by `claude` as a flag, not as its value.**
  `--remote-control`/`--resume` take an optional/positional argument; since both files are agent-writable,
  this turned a one-shot prompt injection into a persistent, unattended reconfiguration applied on every
  restart/reboot (`claude-topic run` execs unattended, under systemd, with no human watching). Both
  values are now validated before they reach a `claude` command line.
- **`AGENT_REPO` resolution picked the alphabetically-first matching directory, silently.** A planted
  directory could shadow the real agent brain repo and take over its registry/display-name. Fixed by
  pinning the glob to the Unix username (`kb-agent-*-"$(id -un)"`) and refusing outright if more than one
  candidate still exists, rather than guessing.
- **`agentic-monitor` carried an independent, unsynced copy of that same resolution — and made a hijack
  worse, not better.** A fix scoped to `claude-topic` alone meant the wrapper would refuse to run (loud)
  while the monitor kept resolving the decoy or nothing at all and reported healthy either way. The
  monitor's per-agent check now uses the identical pinned-glob logic and alarms on all four abnormal
  states (repo missing, ambiguous, resolved-but-no-registry, user-bus unreachable) instead of just one.
- **`agentic-monitor`'s per-agent check ran a login shell (`bash -lc`), which sources the agent's own
  (agent-owned) `~/.profile`/`~/.bash_profile`.** A single planted shell function
  (`systemctl() { return 0; }`) there silently neutralized every health check in that block, regardless
  of how correctly the repo resolved — reproduced on a real dotfile, confirmed byte-identical afterward.
  Switched to `bash --noprofile --norc -c` with an explicit `PATH`.
- Registry keys were never charset-validated before an unquoted `for k in $(...)`; a key of literal `*`
  would glob-expand against the process's working directory. Now filtered through the same charset as key
  validation elsewhere, with a warning for anything dropped.

### Investigated and deliberately NOT done
- **Root-owning the per-agent systemd unit file** (`~/.config/systemd/user/claude-topic@.service`),
  mirroring the root-owned wrapper binary. Proposed, then shown — and independently reproduced — to be
  bypassable via `rm`+recreate, `mv`, or even `sed -i` (which rewrites via rename, not in-place): only the
  *containing directory's* permission gates those, and the agent owns its whole home tree. Shipping it
  would have looked like a closed persistence gap without closing one. The real fix is root-owned
  **system** units (`/etc/systemd/system`) with `User=<agent>` plus a scoped polkit/sudo rule — a bigger
  architectural change, left for a deliberate future decision rather than folded in here.

## [0.4.0] — 2026-07-25 — Three more proven skills

Additive release: the shared layer now ships five fleet-common skills instead of two. Each was written
for, and repeatedly used by, the reference deployment before being generalized here — the rule this
project follows is that a skill earns publication by proving useful, not by seeming like a good idea.

### Added
- **Three more fleet-common skills**, generalized from the reference deployment after they proved
  useful there: **`advisor-review`** (how to get an independent adversarial second opinion out of a
  sub-agent, with the constraints it must always be given and the verify-before-you-act rule),
  **`decision-loop`** (sparring procedure for strategic calls: strawman, attack, converge, record), and
  **`knowledge-governance-workflow`** (OKF frontmatter and hygiene for a brain). `kb-agent-shared/skills/`
  now ships five.

## [0.3.0] — 2026-07-25 — Make the scaffold actually work

A correctness and honesty release. An independent pre-release audit walked the documented bring-up path
end-to-end and found that an adopter finished it with an agent that booted to a deleted file, had no
discoverable skills, an unfilled owner profile and an empty memory directory. Nothing new was worth
shipping on top of that, so this release fixes the scaffold, narrows the claims to what the code
actually does, and adds the one new thing that prevents the same rot returning: a structural linter.

### Added
- **`agentic-divergence-check`** — a read-only daily linter for brain drift: declared shape, adapter
  wiring, folder-skill metadata, symlink resolution, runtime skills registration, live `~/CLAUDE.md`,
  clone-vs-origin agreement, broken references in active docs, and roster-vs-reality. It checks
  **invariants, never technology vocabulary**, so it does not rot — see
  [`docs/divergence-check.md`](docs/divergence-check.md) for why that distinction matters. Drift does
  not page you; it surfaces in the weekly report.
- **`fleet-agents`** — one roster read by every host script, so adding an agent cannot leave it silently
  uncovered by some of them.
- **`skills-archive/`** — retired skills live outside `skills/`, so the runtime stops loading them while
  their reasoning stays in git.
- **`memory-mirror` (optional, timer disabled by default)** — a deterministic, secret-scanned,
  fail-closed mirror of the runtime's working memory into `memory/auto/`. No model involved. Named for
  what it does: the earlier name (`dream`) promised a reflect-and-consolidate pass that was evaluated
  and not adopted, so keeping it would have been the same kind of over-promise this release removes
  from the docs. Commits are prefixed `[mirror]`.
- `memory/README.md`, a `distilled-memory.md` stub and `episodic/` now ship in the brain template.


### Fixed
- **`templates/infra/scripts/provision-agent` now creates the `~/.claude/skills` whole-directory
  symlink.** Without it an adopter's skills were silently invisible to the runtime, contradicting the
  auto-registration promised in `docs/skills.md`.

### Removed
- **The `handoffs/` directory and the file-based handoff channel.** Cross-agent handoffs are now relayed
  by the owner in chat: the sending agent writes the brief as Markdown, the owner pastes it into the
  receiving agent's session. Rationale (documented in `docs/multi-agent-governance.md`): once the shared
  repo is write-restricted to the single privileged agent, a peer-writable handoff directory re-opens a
  cross-agent write channel for a workflow that is low-volume and benefits from the owner seeing every
  brief anyway. Fewer moving parts, one less thing to secure and sync.
- **The `dream` nightly-consolidation script, prompt and units are no longer shipped in
  `templates/infra/`.** The version published in v0.2.0 was an early draft that failed open, granted the
  model `Bash(git *)`, had no secret scan before committing, and cleaned the worktree destructively — not
  something to hand to strangers as root bash. The mechanism is now documented as experimental in
  `docs/memory.md` (reference implementation only). **If you copied it from v0.2.0, do not run it.**

## [0.2.0] — 2026-07-25 — Brain framework v2

### Added
- **Identity split** — `SOUL.md` (durable identity: who the agent is, voice, principles) +
  `OPERATING.md` (operating contract: scope, threat model, gates, bootstrap) replace the single
  `system-prompt.md`. Thin `CLAUDE.md`/`AGENTS.md` adapters load both plus `shared/owner-profile.md`.
- **`shared/owner-profile.md`** — one fleet-wide profile of the owner + org, loaded by every agent, so
  the fleet holds a single consistent understanding of who it serves.
- **Skills v2** — folder-per-skill (`skills/<name>/SKILL.md`, Anthropic Agent Skills format) with
  progressive disclosure, and **whole-directory** harness registration (zero per-skill setup).
  Fleet-common skills live once in `shared/skills/` and are symlinked into each agent.
- **`skillify`** skill — the executable authoring/lifecycle companion to the skills policy.
- **`agent-audit`** skill — an interactive, human-in-the-loop brain tune-up (skills, SOUL, OPERATING,
  memory); the only path that changes an agent's capability or identity.
- **Two-tier memory** — runtime auto-memory (non-canonical working cache) distilled into the Git brain
  (`memory/distilled-memory.md` + `episodic/` + machine-mirrored `auto/`), which is canonical.
- **Nightly "dreaming"** — `infra/scripts/dream` + per-agent `dream@` systemd timer (opt-in, disabled
  by default): a deterministic memory mirror plus a tightly-scoped model run that distils memory and
  appends suggestions to `memory/dream-log.md`. Enforced by the wrapper to touch only `memory/`, so it
  can never alter identity/skills/config. See `docs/memory.md`.
- **Docs** — `docs/skills.md`, `docs/memory.md`; `portability.md`/`architecture.md` updated for the
  identity split.

### Changed
- `policies/skills-policy.md`, `policies/memory-policy.md`, `templates/agent-template.md`,
  `templates/skill-template.md` updated to the v2 model.

### Removed
- `templates/kb-agent-template/system-prompt.md` — superseded by `SOUL.md` + `OPERATING.md`.

## [0.1.0] — 2026-07-24 — Initial framework

### Added
- First public release: per-agent brain repos (`kb-agent-<role>-<name>`) + a shared governance layer
  reached via a committed `shared/` symlink (sibling clone; no submodules); scaffolding templates;
  infra (systemd-supervised Remote Control topics, `kb-sync`, `provision-agent`, monitoring with a
  dead-man's switch); and the docs write-up.

[0.4.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.4.0
[0.3.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.3.0
[0.2.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.2.0
[0.1.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.1.0
