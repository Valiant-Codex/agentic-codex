# Changelog

All notable changes to **agentic-codex** are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/), and the project follows
[Semantic Versioning](https://semver.org/). Pre-1.0: the framework is still evolving, so minor
versions may include structural changes. `1.0.0` is reserved for a deliberate "stable and proven"
milestone.

## [Unreleased]

### Fixed
- **`provision-agent` would happily create a dangling `~/CLAUDE.md`.** It symlinked
  `deploy/home-CLAUDE.md` unconditionally, so a brain repo without that file produced a broken symlink —
  the agent boots with **no bootstrap at all** and nothing anywhere reports it. It now refuses, pointing
  at the seed-the-brain step.

### Added
- **Both bootstrap layouts are now supported**, so this is a per-brain choice rather than a fork:
  *three-layer* (`deploy/home-CLAUDE.md` as the runtime bootstrap plus repo-root `CLAUDE.md`/`AGENTS.md`
  adapters — what `kb-agent-template` ships, worth it if you genuinely run a second runtime) and
  *single-file* (the brain's root `CLAUDE.md` **is** the bootstrap, with absolute paths that resolve from
  any cwd). `deploy/home-CLAUDE.md` still wins when present, so the shipped template is unaffected.
  `agentic-divergence-check` accepted only the first and reported the second as drift; it now accepts
  either while still requiring that `~/CLAUDE.md` point into the agent's own brain repo.
  The reference deployment moved to single-file on 2026-07-29: it runs one runtime, never added
  `AGENTS.md`, and the two files had drifted — paying the duplication cost with none of the portability
  benefit. Either choice is fine; making it by accident is not.

> Working convention: the moment a change is judged **framework-level** (or is a safety/correctness fix,
> which always propagates), its line goes here — before the code is generalized. A release is then: read
> `[Unreleased]`, generalize what it lists, tag. This section existing is the mechanism; an empty one is
> the correct steady state, not an omission.

## [0.5.0] — 2026-07-29 — Provisioning correctness, and stop steering people into a token that grants too much

Everything here was found the hard way: provisioning a **fourth** agent into the reference deployment
surfaced one real bug in `provision-agent`, one place where this repo documented behaviour its own code
does not have, and one recipe that actively told adopters to build the misconfiguration the access model
exists to prevent. Three agents had been provisioned without noticing any of it — the bug only bites on a
user's *first* run, and the token recipe only bites once you check what the token can actually reach.

### Fixed

- **`provision-agent` raced `loginctl enable-linger`, then reported success anyway.** `enable-linger`
  returns once the flag is set; it does not wait for the per-user systemd manager and its D-Bus socket.
  On the first provisioning of a brand-new user, `/run/user/<uid>/bus` therefore did not exist when the
  topic step ran: every `systemctl --user` call died with `Failed to connect to bus`, **no topic was
  enabled**, and the step still printed `[ok]`. It surfaces only on a user's first run — precisely when
  you are provisioning — and any later re-run looks perfect, which is how it stayed hidden across three
  agents. Now: `user@<uid>.service` is started explicitly, the bus socket is polled (30 s cap), and a
  timeout fails hard with the two diagnostic commands.
- **`provision-agent` reported a dead topic as `[ok]`.** The topic loop swallowed errors with `|| true`
  and printed `[ok] topic <key> ($(is-active))`, which rendered as `[ok] topic <key> ()` — an empty
  state presented as success. It now prints the real state, marks anything not `active`/`activating` as
  `[FAIL]`, and exits non-zero with the recovery hint. Verified by reproducing the race on a throwaway
  user (old: `Failed to connect to bus` + `[ok] topic tkey ()`; new: `[ok] linger enabled (user bus
  ready)` + `[FAIL] topic tkey (enable-failed)`, exit 1), then re-checking the success path and
  idempotency against a live agent.
- **`provision-agent-github-access.md` step 5 told you to put `kb-agent-shared` in the root-agent's and
  dev-agent's tokens with Contents: Read and write** — while the access matrix in the very same layer
  gives both of them **R** on that repo. Since a fine-grained PAT applies **one permission set to every
  repository it selects**, following the recipe granted write on the shared governance layer — the one
  grant the model exists to withhold from injection-exposed agents, and the layer every agent auto-loads
  through `kb-sync`. The per-bot repo list now matches the matrix (`kb-agent-shared` appears for the
  sole direct writer only), and it no longer omits the dev-agent's own brain repo, which the matrix
  grants RW.
- **`templates/infra/fleet-agents` described behaviour this repo does not ship.** Its header claimed to
  be the single source of truth for `kb-sync`, `agentic-monitor` and `agentic-divergence-check`; only
  the divergence check reads it — the other two deliberately auto-discover from disk, which is the
  better clone-and-run default. The comment now says which script reads it, why the other two don't, and
  what to change if you want the roster to be authoritative instead.
- CHANGELOG: restored the missing `[0.4.1]` link definition.

### Added

- **An `[Unreleased]` section**, which this project's own governance says is the trigger for propagating
  framework-level change upstream — and which did not exist, so the mechanism had nowhere to write.
- **`manage-agents.md`: a "seed the brain repo" step (a2).** A brand-new brain repo is empty, and
  provisioning an empty repo produces a broken agent *quietly*: `~/CLAUDE.md` becomes a **dangling**
  symlink (the agent boots with no bootstrap at all) and **zero topics** are enabled. The runbook now
  states the four files that must exist before `provision-agent` runs, and why each matters.
- **`manage-agents.md`: an explicit ordering rule for the first interactive login.** Authorize the
  runtime *before* the first topic starts. A topic that first starts unauthenticated registers a
  sessionId that never receives a transcript, after which `claude-topic restart` correctly refuses to
  resume a conversation that does not exist; recovery is `claude-topic restart --new <key>`.
- **`manage-agents.md`: the fleet-roster step (d)**, previously missing from the CREATE flow even though
  `fleet-agents` asks to be kept current — so every new agent left a standing finding in the daily
  divergence report, which is how you train yourself to ignore that report.
- **`manage-agents.md`: sharper validation** — a topic listed `active` with an **empty sessionId** is not
  healthy, plus re-run-idempotency and a divergence-check pass.
- **`github-access-policy.md` §Token type: "a fine-grained PAT inherits nothing".** Neither the org base
  `read` permission nor the bot account's collaborator grants reach a fine-grained token; only its own
  repository selection does. The failure mode is misleading rather than loud: an empty-selection token
  still authenticates (`gh api user` returns the bot) and still lists **public** repos, so it reads as
  working. Documents the only tests that mean anything, and states plainly that for the normal case — RW
  on own brain, R on shared — **a classic PAT is the correct choice, not a compromise**, because it rides
  the account's per-repo grants and therefore matches the matrix for free.
- **`docs/portability.md`: which of the three bootstrap files actually runs.** `deploy/home-CLAUDE.md`
  (loaded always, since the topic unit sets `WorkingDirectory=%h`), the repo-root `CLAUDE.md`, and
  `AGENTS.md` are easy to mistake for duplication. Now spells out who reads each, warns that the
  repo-root adapters are dead weight *that will drift* if you never use a second runtime, and flags the
  specific trap: the root adapter's repo-relative paths do not resolve from `~`, so "consolidating" onto
  it silently breaks every reference.

### Changed

- `provision-agent-github-access.md` is no longer titled and framed around fine-grained tokens; it now
  presents the choice as a deliberate one with a security consequence, and documents the fine-grained
  recipe as the single-repo case.
- **Rotation guidance:** rotate immediately and out-of-band if a token ever appeared in a chat, ticket,
  or transcript — including a conversation with one of these agents. Once a secret is in a transcript it
  is on disk and usually off the box; replace it instead of reasoning about who saw it.
- README: added the comparison against OpenClaw and Hermes Agent (previously committed but unreleased and
  absent from this changelog — "they are runtimes; this is a blueprint").

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

[0.5.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.5.0
[0.4.1]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.4.1
[0.4.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.4.0
[0.3.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.3.0
[0.2.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.2.0
[0.1.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.1.0
