# Changelog

All notable changes to **agentic-codex** are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/), and the project follows
[Semantic Versioning](https://semver.org/). Pre-1.0: the framework is still evolving, so minor
versions may include structural changes. `1.0.0` is reserved for a deliberate "stable and proven"
milestone.

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

[0.2.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.2.0
[0.1.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.1.0
