# Changelog

All notable changes to **agentic-codex** are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/), and the project follows
[Semantic Versioning](https://semver.org/). Pre-1.0: the framework is still evolving, so minor
versions may include structural changes. `1.0.0` is reserved for a deliberate "stable and proven"
milestone.

## [0.8.7] — 2026-08-27 — Skills are validated on shape, never on whether they help

`skillify`'s Validation section checked frontmatter and token count — the shape of a skill, not its
worth. Meanwhile `skills-policy.md` guardrail 8 tells a reviewer to ask "would this still earn its
place against a no-skill baseline", and gave no way to answer it. So the answer was always a guess,
and the guess always favoured keeping the skill: nobody deletes a document that looks fine.

### Added

- **`skillify` Validation now has two levels: shape and effectiveness.** Shape is the old checklist.
  Effectiveness is a procedure: 2–3 realistic test prompts, two sub-agents spawned in the same turn
  (one with the skill's path, one with no skill), same inputs, outputs compared. Three named outcomes,
  each with what it means — including that a *tie* makes a skill a deletion candidate rather than a
  rewrite candidate, which is the case a reviewer is most likely to rationalise away.
- **Triggering is tested separately from quality.** A skill can be good and never fire; the defect
  then lives in `description`, the field the harness actually matches on, not in the body. Testing
  the two together is how a description problem gets misdiagnosed as a content problem.
- **The baseline rule, stated once:** a run *with* the skill proves nothing without the run *without*
  it. When updating an existing skill the baseline is the old version — snapshot before editing.

### Notes

The with/without comparison is borrowed from Anthropic's official `skill-creator` plugin. The
reference deployment **absorbed the idea rather than installing the plugin**, and `skillify` records
why in its `Related` section so the question is not reopened as new: a third-party `SKILL.md` is
instructions a model executes, and in a fleet where one agent holds root, installing one beside our
own puts unreviewed third-party instructions in a privileged session. It would also leave two skills
competing for the same intent — the selection failure guardrail 8 exists to prevent. Ideas from
third-party skills are welcome; the files are not.

## [0.8.6] — 2026-08-25 — Voice moves to output style; a CHANGELOG a checker cannot parse is not clean

A reference-deployment change (Durin + Galadriel both moving their voice/register out of `CLAUDE.md`
prose into Claude Code's `outputStyle` mechanism) needed real support here first: `provision-agent` had
none, and the OKF frontmatter checker would have flagged the new files as violations the moment anyone
tried. Bundled with one propagation gap the new codex-propagation-check caught the same day.

### Added

- **`provision-agent` symlinks an output style, if the brain declares one.** `deploy/output-styles/<agent>.md`
  → `~/.claude/output-styles/<agent>.md`, same live-symlink treatment as `CLAUDE.md` and `skills/` —
  not the real-copy treatment `claude-settings.json` gets, because an output style carries no
  permission grant, so the F1/F3 reasoning for keeping settings from live-following the repo does not
  apply to it. Optional and non-fatal: a brain without one is skipped, not failed.
- **A third OKF frontmatter exemption: `deploy/output-styles/*.md`.** That frontmatter is Claude Code's
  own schema (`name`, `description`, `keep-coding-instructions`), not OKF's, parsed live by the harness
  the moment the file is written — same reasoning as the existing `CLAUDE.md` exemption. Documented in
  `knowledge-governance-workflow/SKILL.md`; `agentic-divergence-check` now skips the whole
  `output-styles/` directory during its frontmatter walk, the same way it already skips `auto/`.
- `deploy/README.md` in the agent template documents the new optional `output-styles/<agent>.md` file
  alongside the existing `claude-settings.json` row.

### Fixed

- **Ported yesterday's guard, one day late.** `codex-propagation-check` compares the set of conditions
  each script refuses to pass and found this repo had not carried a guard added to the reference
  deployment the day before: a `CHANGELOG.md` that exists but yields no parseable version head is
  cannot-tell, not nothing-to-report. An absent tag legitimately means "this repo does not tag"; an
  unparseable present file does not. That is the whole point of the check the guard belongs to.

## [0.8.5] — 2026-08-25 — The lint shipped yesterday certified fail-opens as safe

0.8.4 shipped `failopen-lint` and said the number it reports is the point. An independent review of
that release found the lint was **exempting the shape it exists to find**, and had already retired two
real fail-opens from the backlog it was measuring. If you took 0.8.4, take this.

### Fixed — the lint itself

- **`&&` must never exempt.** The `HANDLED` pattern accepted `&& drift` alongside `|| drift`. They are
  opposites: `cmd || drift` reports on the failure branch, while `[ -n "$x" ] && drift` reports only on
  non-empty — and *empty is exactly what a failed command produces*. A three-line fixture containing
  nothing but the unsafe shape was reported "1 explained, 0 UNEXPLAINED", exit 0. A lint that blesses
  its own quarry is worse than no lint: the number shrinks and gets trusted. Only `||` exempts now, and
  the reference deployment's honest count went back from 25 to 30.

### Fixed — the two it had hidden, plus three of the same family

- **A repo whose `git status` fails read as clean**, in the strict-clean assertion whose job is
  catching root-installed code left local-only. `porcelain="$(git status --porcelain 2>/dev/null)"`
  followed by `[ -n "$porcelain" ] && drift`. The same shape had been fixed fifteen lines above in
  0.8.4 and was invisible here because the loosened lint had just excused it.
- **An unwalkable tree read as "no dangling symlinks."**
- **`${UPDATE_HC_URL:?…}` exits 1, and the drift-check unit sets `SuccessExitStatus=1`** — so the guard
  documented as FATAL was recorded by systemd as a *successful run* for its likeliest trigger, someone
  editing the env file. The missing-file branch exits 2 and was genuinely fatal; the unset-variable
  branch was not. The `:?` idiom is correct in the monitor script, which has no such setting, and was
  copied across a different contract. Now an explicit `exit 2`.
- **`memory-mirror` gained a new instance of the class 0.8.4 fixed in it.** A failed
  `git show HEAD~1:memory/auto` also yields an empty pipeline and therefore `0`, indistinguishable from
  "the previous commit held no memories" — silently disabling the drop guard and making the
  `''|unknown` case arm dead code. It now asks whether git could answer before counting.
- **`|| echo 0` survived in `install-host-services` and `provision-agent`**, in the guards that refuse
  to install unreviewed root-owned code from an unpushed repo. A failed `rev-list` read as "nothing
  unpushed". Both now fail the guard.

### The lesson worth more than the patches

0.8.3 closed three fail-opens by hand. 0.8.4 found six more and built a lint because hand-enumeration
does not converge. 0.8.5 exists because **the lint was written by the same author, in the same pass,
and inherited the same blind spot** — and the only thing that caught it was an independent reviewer
running a deliberately broken fixture through it. A tool that measures your work is still your work.
Point someone else's fixture at it.

## [0.8.4] — 2026-08-24 — Six more fail-opens, and a lint so the next six are found by a machine

0.8.3 hunted this class by hand and said a clean run proves nothing. Then a cross-cutting audit found
six more instances that had survived that very pass — including one inside the frontmatter checker
itself. The lesson is not "look harder next time": **enumerating fail-opens by reading does not
converge**, so this release ships the mechanical check first and the fixes second.

### Added

- **`scripts/failopen-lint`** — vocabulary-free, knows shell shapes and not any deployment's nouns.
  Flags three constructs in which *I could not tell* renders as *I looked and it was fine*: `|| true`
  swallowing an exit code, `2>/dev/null` feeding a count, and a loop over a variable that may be
  empty. A line is exempt only if a comment within three lines says why empty is legitimately clean,
  or if the failure is handled on the spot with `|| drift`. Either drift, or write down why not.
  It reports a number; the number is the point, not zero.

### Fixed — fail-opens

- **Broken `apt`, `docker` or `curl` reported a patched, current box and exited 0.** A failed
  `apt list --upgradable` gave `grep -c` empty stdin, so "0 upgradable (0 security)" was
  indistinguishable from fully patched; an empty running-version or latest-release string fell
  through to the else and left the update counter at zero. Exit 0 also pings the dead-man's switch
  as OK — so the security-update detector was dead behind a green heartbeat. Both now drift.
  Proven with stub binaries exiting non-zero, old code against new, side by side.
- **A missing monitor config made every drift finding silent.** Sourcing the config file had no
  `else`, and the report ping was guarded by `[ -n "${UPDATE_HC_URL:-}" ]`. With the file absent,
  drift was printed, never delivered, and — because the unit sets `SuccessExitStatus=1` — the unit
  did not even look failed. The sibling monitor script had guarded its own URL this way since it was
  written. Now fatal, exit 2. Delivery outcome is also reported, because an undelivered report used
  to be indistinguishable from a delivered one.
- **`git rev-list --count … || echo 0` made a failed query read as "in sync"**, at two call sites.
  Empty now means the command failed, and that drifts.
- **A CHANGELOG that exists but yields no parseable version head was "nothing to report".** An absent
  tag legitimately means "this repo does not tag"; an unparseable present file is cannot-tell.
- **The frontmatter checker skipped any file with no frontmatter block at all** — so carrying no
  metadata whatsoever was the one way to pass every metadata assertion. That is this checker's own
  hunted shape, sitting inside the checker. An unreadable file was also `continue`, and now drifts.
  Closing it immediately surfaced findings in **archived, read-only** repos, which no one can fix; so
  frontmatter now skips dormant brains, for the same reason the reference sweep already does. A
  permanent unactionable finding is how an alarm channel stops being read. Proven both ways.
- **The memory mirror's drop guard counted two different sets** and cried wolf on a healthy fleet:
  the current count excluded the index file, the previous count included it, so the guard fired every
  night the memory count merely held steady. On the reference deployment it held the alarm channel in
  failure for nine hours over a rename. Same line also fixed a `grep -c … || echo unknown` that
  produced a two-line string matching no case arm, erroring out with no finding and no exit code.

### Fixed — coverage

- **`install-host-services` now installs itself and `provision-agent`**, via an atomic rename rather
  than in-place `install` (overwriting a running script is how a shell reads garbage). Until now they
  were the only root-installed code that appeared in no manifest row, so the "installed matches
  source" sweep could not see them by construction. On the reference deployment the installed copy
  sat **15 days behind git**, missing the commit that stops `--enable-writers` from re-enabling a
  dormant agent's nightly writer against an archived repo — and the daily check reported zero drift
  throughout.

### Note on completeness

`failopen-lint` reports 25–29 remaining unexplained shapes in `agentic-divergence-check`, mostly
`while read` loops and their producers where an empty producer means "nothing to check" and the thing
being checked drifts elsewhere. **They are deliberately not annotated as safe.** Writing "this one is
legitimate" without verifying each is how a fail-open becomes permanent with a written blessing. The
number is a measured backlog, not a claim of completeness.

## [0.8.3] — 2026-08-23 — A guardian that cannot tell must say so

Three fail-opens in the drift checker, a memory-durability gap nothing would have reported, and a
budget that measured the cheaper half. Found by auditing the checker against deliberately broken
fixtures rather than trusting its clean runs — which is the practice this release is really about:
**a clean run never proves a check fires.**

### Fixed — fail-opens

- **PyYAML missing reported CLEAN, and deceptively.** The frontmatter validation ran twice: a verbose
  pass that printed `[DRIFT]` lines and always exited 0, and a silent pass that returned the count.
  Only the second fed the failure counter, so with PyYAML absent the log looked dirty while the exit
  code and the dead-man's-switch ping both said clean. The comment above it claimed "Fails CLOSED".
  Now one pass, reporting on a `COUNT=` line: the walk and the `type` vocabulary exist once, because
  two copies of the same logic is how they drift apart.
- **The reference check reported clean when it could not list files.** `subprocess.run` does not raise
  on a non-zero exit, so an unreadable repo returned an empty file list — indistinguishable from a
  repo with nothing wrong. It now drifts on a failed `git ls-files`, and on a repo with no `.md` at
  all. *I could not look* must never render as *I looked and it was fine*.
- **The AGENT_PATH assertion could not fail if the files were missing.** It counted the distinct
  values it found, so three missing files plus one present was one variant — "unanimous". Only the
  0-of-4 case drifted. It now asserts existence first: absence is not agreement.
- **`${VAR:-default}` became `${VAR-default}`** for the roster, workspace and strict-clean lists. An
  operator who empties one of these means empty; the `:-` form silently restores the built-in, which
  is how a deliberately cleared list becomes last month's again.

### Fixed — memory durability

- **`memory-mirror` asserted a great deal about its own execution and nothing about its result.** Two
  ways memory is lost that nothing would have noticed: a file with no line in `MEMORY.md` is invisible
  to recall (the bytes are safe and the fact is gone, which is worse than an error because it looks
  healthy), and `rsync --delete` propagates a disappearance from the store into the repo under a
  generic `[mirror]` commit nobody reads. Both now raise the exit code, which routes to the existing
  failed-unit alarm.

  **They run after the push and never abort — that ordering is the design.** The secret scan aborts
  *before* mirroring on purpose, because a secret in git cannot be un-published. This is the opposite
  case: aborting on a coverage gap would turn a findability problem into a durability outage, so the
  check would cause a worse loss than the one it prevents.

### Added

- **Dormant agents named as live actors.** The checker had no way to see the class of staleness that
  matters most after an agent is put to sleep. Names are **derived** from the roster and
  `fleet-dormant`, so waking or adding an agent needs no edit; the only literal list is a handful of
  English tense markers, which describe grammar rather than technology and therefore do not rot.
  `decisions/`, `memory/` and `archive/` are exempt — a record that cannot name a dormant agent is
  useless. On its first run in the reference deployment it found ten live references that a full day
  of manual sweeping had missed, including an access matrix still granting a revoked bot write on
  production.
- **The always-on budget now counts the runtime auto-memory index.** It watched `CLAUDE.md` plus its
  imports and ignored the index the runtime injects on every session — which lives outside the repo
  and, measured, cost more than the layer being watched. The path is derived from the agent's home,
  the threshold is `CONTEXT_BUDGET`, and the breakdown is printed on **every** run, not only on
  breach: the index grows by normal operation, so the useful signal is the trend.
- `CONF` is overridable, so the checker can be exercised without pinging the dead-man's switch.

### A note on the budget message

The first version of the breach message said "trim the memory index". That prescribes the exact way a
memory is lost: an unindexed file is unreachable. It now says to shorten the hooks, and says why
deleting a line is a loss rather than a saving. Worth stating because it is the general failure —
**a check whose remedy causes the harm it was written to prevent.**

## [0.8.2] — 2026-08-22 — Dormancy is a state the scripts have to agree on

A fleet that puts an agent to sleep needs *both* root-installed scripts to know it. In the reference
deployment they disagreed: `agentic-divergence-check` asserted a dormant agent's nightly writer is
**off**, while `install-host-services --enable-writers` enabled it for every name in the roster. On a
rebuilt box — the flag's documented use case, and the moment nobody is watching — that resurrects a
writer pushing to an archived repo: 403, failed unit, a page every night. An unattended writer turning
itself back on is exactly the blast radius a human-confirm gate exists to bound.

*(By this project's own rule a batch of correctness fixes is a minor. Shipped as a patch on the
maintainer's call; the substance is below either way.)*

### Added
- **`templates/infra/fleet-dormant`** — one Unix user per line, next to `fleet-agents`. A dormant agent
  keeps its user, clones and roster entry (removing it would leave an unswept home nothing looks at),
  so dormancy needs its own declaration rather than an absence. Read by the installer *and* the drift
  check, which is the point of having a file instead of a literal in one script.
- **The dormant-brain bail-out in `templates/infra/scripts/agentic-divergence-check`**, which had
  landed in the reference implementation and not here. A dormant brain is a frozen snapshot: its prose
  points at a world that moved on, and fixing references in a repo nobody may write buys nothing.
  Everything before the bail-out still applies — including that its `CLAUDE.md` must say DORMANT.

### Fixed
- **`install-host-services --enable-writers` now skips dormant agents.** Two details found by testing
  fixtures rather than reading the code, both in the new code: a `fleet-dormant` containing only
  comments made `grep` exit 1 and, under `set -euo pipefail`, aborted the whole installer — and an
  empty list is the normal state; and resolving it with `${VAR:-default}` meant a deliberately emptied
  file silently fell back to a hardcoded list. The file is authoritative **when it exists, empty
  included**; only a missing file uses a built-in, and it says so.
- **`README_AGENT.md` claimed `owner-profile.md` is "loaded every session".** It is not — only
  `CLAUDE.md` is auto-loaded. This is the precise false claim that release 0.7.0 existed to eliminate
  (every `SOUL.md` asserted it in its own frontmatter and it was false the day it was written),
  reintroduced in the one document a new adopter follows literally.
- **`docs/architecture.md` had two bullets for `CLAUDE.md`**, the second reading "the single runtime
  bootstrap … that points to **them**" — a dangling referent left by deleting `SOUL.md` and
  `OPERATING.md`.
- **The prerequisites omitted `python3-yaml`.** `apt install python3` does not provide PyYAML, and
  without it the frontmatter validation cannot run — so a new adopter's very first check reports a
  drift finding per brain. (The deeper issue, that this path reports CLEAN instead of failing closed,
  is a known open item and not fixed here.)
- **`app-layer.md` said the secret store is "backed up off the box, regularly"** while
  `reference-architecture.md` says plainly that an off-box export is "the documented open item — not
  yet the achieved state". Two pages of one repo disagreeing on a security fact, with the optimistic
  one being what an adopter reads while planning recovery. Aligned to the honest version.
- **`runbooks/` was still listed as part of the shared governance repo in three places**, abolished in
  0.7.0 — a claim the repo already contradicted in `templates/root-agent-skills/README.md`
  ("That directory is gone").
- **`docs/divergence-check.md` described the timer assertion in the direction that inverts for dormant
  agents**, and `DORMANT_AGENTS` appeared nowhere in the documentation.
- Reference-architecture and app-layer now describe **two** active agents and the unprivileged build
  user that runs the npm toolchain, matching what the reference deployment actually does since its
  dev-agent was folded into its root-agent.

## [0.8.1] — 2026-08-18 — Take the ladder, not the plugin

Coding agents over-build. That is not a controversial claim, and a viral MIT-licensed plugin —
[ponytail](https://github.com/DietrichGebert/ponytail), 104k stars in two months — packages a good
correction for it: a seven-rung ladder you climb before writing code, stopping at the first rung that
holds. The correction is worth having. The **packaging** is not, and that gap is the whole point of this
release: **an agent brain takes on prose, not dependencies.**

### Added
- **A `Build discipline` section in the agent template**
  ([`templates/kb-agent-template/CLAUDE.md`](templates/kb-agent-template/CLAUDE.md)), marked for
  deletion on agents that do not build — it earns its ~350 tokens only where the agent writes code.
  Seven rungs: does this need to exist at all, is it already in these repos, stdlib, native platform
  feature, already-installed dependency, one line, and only then the minimum that works. Plus the two
  rules that keep it from becoming a licence to be careless — the ladder runs **after** you understand
  the problem, never instead of it, and it never simplifies away trust-boundary validation, error
  handling that prevents data loss, security or accessibility.
- **[`app-layer.md`](docs/app-layer.md) records why the ladder is copied and the plugin is not.** Three
  reasons, each of which generalizes well past this one plugin:
  - its hooks execute `node` at `SessionStart`, `SubagentStart` and `UserPromptSubmit`, with the
    agent's own privileges, updated through a plugin marketplace nobody on your side reviews — on a box
    where one agent holds sudo, that is a standing code-execution channel into the privileged context;
  - its `SubagentStart` hook ships with **no matcher**, so "at most three short lines, no essays" lands
    in code-review, audit and security subagents too, and a reviewer told to be brief reports fewer
    findings;
  - the advertised token saving does not survive independent measurement — roughly −10% with an
    interval that touches zero, against −20% advertised — which matters precisely because the saving is
    the reason most people install it.

The mechanism for this was already here: one always-on contract file, per agent, since 0.7.0. What was
missing was the worked example of using it to absorb a third-party idea **without** taking on the third
party. That is the shape worth repeating: read the plugin, keep the judgement, leave the hooks.

## [0.8.0] — 2026-08-16 — The capability was already there; nobody had told the agents

A framework can ship a capability and still not deliver it. The `claude-topic` wrapper has been
root-owned, installed for every agent, and scoped to the caller's own brain repo since long before this
release — any agent could always manage its own sessions. But only the privileged agent's
`manage-agents` documented it, so in the reference deployment the unprivileged agents never reached for
it, and every "create me a topic" went through root. **The gap was not permission. It was knowing.**

### Added
- **[`topic-management`](templates/kb-agent-shared/skills/topic-management/SKILL.md)** — the sixth
  fleet-common skill. Each agent manages **its own** topic sessions; cross-agent work goes to the
  privileged agent. That split is enforced by the wrapper itself, not by convention: it resolves
  `kb-agent-*-$(id -un)`, so an agent cannot address another's topics even by accident.

  The skill exists for the judgement around the commands, not the commands:

  - **The wrapper has no self-guard**, and this is the finding that motivated writing it down. Nothing
    stops an agent from stopping or removing the very topic its session runs in — and the process
    executing the command is the one being killed. For `remove` it can die *after* disabling the unit
    and *before* committing the registry. The check is one line, and it belongs in the reader's head
    before any mutating verb: `sed -n 's|.*/claude-topic@\([^/]*\)\.service.*|\1|p' /proc/self/cgroup`.
  - **`remove` is a human-confirm gate; `stop` is usually what was meant.** Removal leaves the
    transcript on disk but nothing resumes it — practically, the conversation is gone.
  - **A new topic is a permanently-on session**, reachable from the web and mobile apps and restarted
    on boot, costing context and money continuously rather than per use. Say that out loud before
    creating one.

  Note what this release does **not** do: it adds no code, no privilege, and no new surface. An
  injection into an unprivileged agent's session could already have run the wrapper — the binary was
  always there and always executable. Documentation that only the root agent holds is not a security
  boundary; it is just a bottleneck that looks like one.

### Fixed
- **`manage-agents` carried a second, drifted description of the same wrapper.** Its command list had
  fallen behind the tool it documents — missing `remove`, `urls`, `rotate` and `rotate-all`. Two
  descriptions of one mechanism drift; there is now one, in the shared skill, and the root-agent skill
  keeps only what is genuinely privileged: reaching *another* agent's topics by becoming that agent.
- **Two silent-emptiness traps recorded** in `manage-agents`, both of which returned success while
  showing nothing on the reference deployment. Running the wrapper as another agent needs **both** `-H`
  (or `$HOME` resolves to the caller's, finding the wrong brain repo or none) **and**
  `XDG_RUNTIME_DIR` (or the target's systemd user manager is unreachable). And a shell glob under
  `sudo` — `sudo ls /home/<AGENT>/.config/systemd/user/*.target.wants/` — is expanded by *your* shell
  before sudo runs, so an unreadable directory yields no match at all, which reads as "nothing is
  enabled" when in fact everything is.

## [0.7.2] — 2026-08-16 — Measure the context before you optimize it

0.7.0 fixed a layer that was never loaded. This release is about the opposite mistake: **spending
effort on the layer that loads but barely weighs anything**, while the real cost sits somewhere nobody
was looking.

### Added
- **[`docs/context-budget.md`](docs/context-budget.md)** — what a session actually loads, measured on
  the reference deployment rather than estimated, with the method to re-measure it yourself.

  The headline: **your own content is 11–20% of the budget.** The agent contract, the memory index and
  the skill listings together came to ~4,200 tokens of a ~38,000-token session. An aggressive rewrite
  of every contract would have recovered ~1,000 tokens; two configuration changes recovered ~15,000.

  The surprise that pays for the page: **MCP tool *schemas* are deferred, tool *names* are not.** Every
  configured server's full tool list is always-on at ~16 tokens per tool, paid every session whether or
  not the server is touched. One ~546-tool infrastructure MCP measured **8,824 tokens — 23% of an
  entire session**. A single built-in tool (`Workflow`) measured ~4,900 on its own, more than that
  agent's contract and memory index combined.

  Also documented: a *failed* MCP server costs no context but still attempts a connection every
  session; two servers exposing identically-named tools is a selection problem (one agent here ran an
  Atlassian MCP whose 31 tools were a strict subset of a connector's 40); and `claude doctor` reports
  installation health only — nothing about context, so do not go looking there.

- **`mcp-park`** ([`templates/root-agent-skills/manage-agents/scripts/mcp-park`](templates/root-agent-skills/manage-agents/scripts/mcp-park))
  — attach or park an MCP server on demand. Moves the definition between the live config and a parked
  file, both `0600` and outside git, atomically, without printing the credentials it carries; verified
  byte-identical across a round trip. A session reads MCP config at startup, so it offers `--restart`
  and prints the reachability warning a restart now always deserves.

### Fixed
- **The `.mcp.json` in a brain repo is decorative, and the docs said otherwise.** Where the supervised
  session runs with `WorkingDirectory=%h` — cwd is the agent's *home*, not its brain repo — that file
  is out of project scope and **is never read**. It is not installed by `provision-agent` and not passed
  by the session wrapper. On the reference deployment it had drifted from reality in all three brains
  (one listed a parked server, another was missing two running ones) and nothing noticed, because
  nothing read it.

  Corrected in `docs/secrets.md`, `docs/portability.md`, both repo-shape templates and the
  `manage-agents` reference. MCP configuration now appears in portability's **not carried** table,
  where it belongs: it lives in the runtime's user-scoped config, alongside credentials, outside git.
  If you want it portable, pass it explicitly with `--mcp-config` from the supervising unit — and then
  check it, rather than assuming a file in a repo is doing something.

- `skills-policy.md` extends its shadowing guardrail to MCP tools, which suffer it worse: unlike a
  skill, every tool name is always-on.

## [0.7.1] — 2026-08-11 — One contract, one index, and a shipped broken link

0.7.0 consolidated the *always-on* layer and left the *governance* layer alone. An independent
fresh-context audit of the reference implementation found what that omission cost, and every finding
had already propagated here. Nothing in this release is behavioural: it is duplication removed,
indexes made honest, and one contract that contradicted itself.

**The frontmatter contract shipped in three places.** `skills/knowledge-governance-workflow/SKILL.md`,
`kb-agent-shared/README.md`, and `kb-agent-shared/templates/README.md` each carried a copy. Only the
first is asserted by `agentic-divergence-check`; both others had drifted back to the pre-consolidation
required set (`type` + `title` + `status` + `timestamp`) and asked for a `type` that was "concept kind,
lowercase" with **no vocabulary at all** — the precise drift 0.7.0's closed nine-value list exists to
prevent. They are also the two files a new document is scaffolded against, so they were the copies
that reproduced. Both now point at the skill, which states outright that it is the only copy and that
anything else must link rather than restate.

**`knowledge-governance-workflow` contradicted itself on `type: skill`.** The vocabulary table scoped
it to `skills/<name>/SKILL.md`; the Validation checklist said "`skill` under any `skills/` folder";
the Related section said "anything under `skills/`". Under the broader reading, every skill's own
`references/*.md` is a violation — which would make progressive disclosure, the thing that keeps a
`SKILL.md` small, non-conformant by construction. Now scoped to the file in all three places, with
the `reference` row saying so explicitly. Recorded alongside it: the checker asserts the vocabulary
and **not** location, so the two rules in that checklist have different teeth — one fails a run, the
other is honour-system. A contract that does not say which of its rules are enforced invites you to
trust the wrong half.

**A broken index shipped to anyone who cloned this.** `kb-agent-shared/reference/README.md` indexed
exactly one file, `claude-code-runtime.md`, which was never shipped here at all — so `reference/`
arrived as a directory whose only content was a README pointing at nothing. The folder is removed.
Stable background material belongs at the root (`owner-profile.md`); the `reference` type keeps its
place in the vocabulary.

**Removed:**
- `kb-agent-shared/policies/operating-principles.md` — 15 principles with no mechanism, referenced by
  nothing, each already binding in a policy, a decision record, a skill, or each README's own
  Maintenance Rule. Its own principle 14 ("keep folder READMEs current") was being broken by three
  READMEs at once, which is the clearest statement available of what an unmechanised principle is
  worth. The one principle with no other home — weigh licensing, maintainability, portability and fit
  before adopting a tool — survives as `decision-loop` step 8, where a real decision passes through it.
- `kb-agent-shared/templates/index.md` — a second index of an already-indexed directory, described by
  its own sibling README as "historical/secondary" and typed with a value outside the vocabulary.
- `kb-agent-shared/reference/` — see above.

**Indexes that were lying:**
- `kb-agent-shared/README.md` and `index.md` both omitted `skills/`. That is the one directory whose
  contents reach every agent's context — including the privileged agent's — through kb-sync, without
  review. Of everything a navigation map can forget, it is the worst choice. Added to both, labelled
  with why it matters.
- `kb-agent-shared/README.md` listed `CLAUDE.md` **twice** in the agent-repo shape, one line still
  calling it a "bootstrap pointer" — a half-applied 0.7.0 edit shipped as the canonical diagram of the
  layout 0.7.0 introduced. It also omitted `deploy/`, the Tier-3 payload. Both fixed, with per-line
  comments so the diagram now explains itself.
- `kb-agent-shared/skills/README.md` still described `agent-audit` as auditing "SOUL, OPERATING,
  memory" — concepts 0.7.0 abolished, in the description of the skill whose job is catching exactly
  this.
- `kb-agent-shared/policies/README.md` typed three rows with invented values (`approval-policy`,
  `memory-policy`, `operating-principles`). Invisible to the checker, which parses frontmatter and not
  prose tables — a reminder that a closed vocabulary only closes where something reads.
- `docs/divergence-check.md` claimed the check verifies "the adapter actually loads SOUL + OPERATING +
  `shared/owner-profile.md`" — inside the same bullet that says no `SOUL.md`/`OPERATING.md` may
  survive. It named a mechanism 0.7.0 dropped and a check that never existed in any version. Replaced
  with what the check actually asserts about frontmatter, including that it fails closed when PyYAML
  is missing and that it never checks `type` against location.

**Upgrading:** delete the three paths above from your `kb-agent-shared`, and if you have restated the
frontmatter contract in a README of your own, replace it with a link. No script, unit file, or brain
layout changes.

## [0.7.0] — 2026-08-07 — One always-on file, because only one file was ever loaded

**Breaking for the brain layout.** `SOUL.md` and `OPERATING.md` are gone. Each agent has one always-on
file, `CLAUDE.md`, carrying identity and voice, scope and delegation, the untrusted-content directive,
the human-confirm gates, the autonomous-OK list, the threat model and the safety invariants.

### Why

The v2 split (0.5.0) divided identity by *durability*: `SOUL.md` for who the agent is, `OPERATING.md`
for what it does, `CLAUDE.md` a thin pointer at both. The reasoning was sound. The behaviour was not
what anyone assumed.

An audit of the reference deployment traced the loading path and found there is none. No `@`-import, no
hook, no `~/.claude/CLAUDE.md`, no `--append-system-prompt`, nothing in the topic unit. **`CLAUDE.md` is
the only file the runtime loads by itself** (plus each skill's `name`/`description`, via the
`~/.claude/skills` whole-directory symlink). The other two entered context only when the model chose to
obey an instruction telling it to read them. Measured on that deployment's own transcripts, over
substantial sessions:

| agent | read `SOUL.md` | read `OPERATING.md` |
|---|---|---|
| root agent | 37% | 54% |
| CoS agent | 16% | 22% |
| dev agent | 47% | 53% |

Meanwhile **45–70% of each `OPERATING.md` was content that must bind before the agent knows to go
looking**: the confirm gates, the threat model, the delegation map, the never-commit-secrets invariant,
and — for the privileged agent — the fact that it is effectively root. The brakes were out of context in
the majority of sessions. Every `SOUL.md` even asserted `Loaded every session alongside OPERATING.md` in
its own frontmatter, which was false the day it was written.

**The general lesson, which outlives this framework: a layer that loads only by instruction is not a
layer, it is a suggestion.** If a rule must hold, it belongs in the file the runtime reads by itself.
Splitting by durability is a good instinct for authoring and a bad one for loading — git already records
what changes rarely and what changes often, for free, without a second file to keep in sync.

Size is not the constraint people assume. Verified against Claude Code 2.1.224: a `CLAUDE.md` is skipped
only above **4 MiB**, with a soft threshold at **40k characters**. A complete single-file contract runs
8–12 KB. The real ceiling is attention, so 0.7.0 budgets **20 KB** and the drift checker enforces it.

### Migrating from 0.6.x

1. Fold `SOUL.md` and `OPERATING.md` into `CLAUDE.md`. Keep everything that must bind before the agent
   knows to look; move the rest — link lists, path mechanics, historical asides — into a skill or drop it.
2. Delete both files. Git holds them.
3. Update the drift checker (shipped here) — it now asserts the one file exists, carries the gates, the
   untrusted-content rule and the autonomous-OK list, and stays under 20 KB, and reports a surviving
   `SOUL.md`/`OPERATING.md` as drift.
4. If you genuinely need a second file, use `@`-imports (they resolve, to depth 5) rather than an
   instruction telling the model to go read something.

### Also removed, for the same reason

- **`runbooks/` is abolished.** The runtime discovers skills and discovers nothing else, so a procedure
  filed anywhere else is reachable only if some already-loaded text happens to name its path — which,
  measured, one skill in nine ever did. Depth belongs in a skill's own `references/`. The framework's
  privileged procedures now ship as **`templates/root-agent-skills/`** (manage-agents, patch-management,
  fleet-brain-change, fleet-monitoring), to be copied into whichever agent you make privileged — not into
  the shared layer, where every agent would carry a lifecycle procedure only one may run.
- **`skills-archive/` is abolished.** Retire a skill with a banner or delete it; git holds the reasoning.
  In the reference deployment the archive directory had accumulated six live references to an archived
  skill, three with broken paths, while its own registry still listed it as active.
- **`memory/episodic/` is dropped.** Specified since 0.2.0, instantiated by one agent, abandoned after
  four weeks, and referenced by five repos including one describing a directory that never existed. The
  runtime records incidents by itself and the nightly mirror carries them.

### Governance

- **`policies/skills-policy.md` is now the single source of truth for skill format.** It records what
  the runtime *actually* enforces (verified in the binary): skills must be directories containing
  `SKILL.md` — a bare `.md` is discarded before it is read; **the invocable name is the directory name**,
  not frontmatter `name`; `name` and `description` are optional to the runtime and required by us; 128 KB
  cap. The status enum is settled at four values (`active | draft | superseded | archived`) — it was
  three in some documents and four in others, while a shipped skill used `draft`.
- **The OKF `type` vocabulary is closed** at 9 values, down from 21 observed — and, unlike a rule that is merely written down, it is now asserted by the drift checker. One deployment had reached
  seven different types for "README of a folder", and a 16/12 split between `decision` and
  `decision-record` with no rule distinguishing them.
- **Two OKF exemptions are finally written down**: `CLAUDE.md` (a runtime artefact — frontmatter there is
  noise) and `memory/auto/**` (byte-identical machine mirrors whose own writer emits unquoted colons).
  Without them, 31% of a conformant fleet "violated" a rule never meant to reach it.
- **`agentic-divergence-check` validates YAML frontmatter**, with those two exemptions, failing closed if
  PyYAML is absent. The governance had required parseable frontmatter since 0.4.x and nothing checked it.
- **Dormant agents are held to a dormant contract**: the untrusted-content rule and an unmissable dormancy
  statement, not gates and an autonomous-OK list describing work they must not do.


### Fixed after release (same day)

An adversarial audit of the 0.7.0 migration found the same defect **25 times**, always the same shape:
**a concept abolished in the file that defines it, and left standing in every file that references
it.** Root cause of the misses: in that environment `grep` was a shell function wrapping `ugrep` with
`--ignore-files`, under-reporting by ~5×. Worth knowing if you audit your own migration.

The corrections, all shipped under this version:
- `bootstrap.md` still instructed every agent to read `SOUL.md` and `OPERATING.md` — item 4 in every
  read list, so the most-read stale instruction in the fleet.
- `skillify` and `docs/skills.md` still taught `skills-archive/`, contradicting the policy the same
  release declared authoritative — in the skill an agent invokes to retire a skill.
- `agent-audit` was half-migrated: step 1 loaded `CLAUDE.md` while passes 2 and 3 were still titled
  *SOUL* and *OPERATING*. That skill is the mechanism that would otherwise catch the rest of this list.
- The closed `type` vocabulary was **declared and not applied**: 36 files still carried retired values,
  including every `decision-record` the doc claimed had been consolidated. Now applied — and asserted
  by the drift checker, because a vocabulary nothing checks is a preference.
- `templates/kb-agent-shared/policies/github-access-policy.md` gave **write on the shared governance
  layer to the most injection-exposed agent**, then two paragraphs later asserted the opposite. The
  matrix now matches its own reasoning: the least-exposed principal holds that write.
- The public checker's frontmatter assertion on the governance layer was **dead code** — it gated on
  the roster's first line, which is a comment, so it never ran and reported clean.
- The public checker was missing the CHANGELOG-vs-tag and `@`-import-budget assertions.
- The seed template told a new agent three false things, including that its distilled memory is
  "loaded every session" while the README beside it said the opposite.
- Internal deployment names had survived under `templates/`, against `CONTRIBUTING.md`'s own rule.

### Added after release
- **`@`-imports count toward the always-on budget**, and an import resolving *outside* the project root
  is reported as drift: it hits Claude Code's external-includes approval gate, which a non-interactive
  supervised session cannot answer — so it would silently fail to load, with no error anywhere.
  Verified empirically: an in-tree `@memory/distilled-memory.md` inlines (839 tokens measured, and the
  model can quote the file with all tools denied); resolution is relative to the **resolved** path of
  `CLAUDE.md`, i.e. the repo root, not `~`.

## [0.6.8] — 2026-08-07 — Two checks that were nagging about the wrong thing

Applying 0.6.7's own findings to the reference deployment made the drift checker start reporting
daily — which is how both of these surfaced. A check that cries wolf is worse than no check, because
the fleet learns to skim past it.

### Fixed
- **`agentic-divergence-check` reported `claude-topic-rotate-on-boot.service` as undeclared drift for
  every agent, every day.** The "user unit enabled but not declared in `deploy/`" rule already
  excluded `claude-topic@` — provision-agent installs it from infra, so it is *fleet code, not
  per-agent config*, and its absence from `deploy/` is correct rather than drift. The rotate-on-boot
  unit is the same class; it landed in 0.6.5 and the exclusion was never extended. Every deployment
  running 0.6.5+ has been getting this false positive once per agent per day.
- **The memory-mirror assertion had no concept of a dormant agent.** It required the timer enabled
  for every agent in the roster — correct for a live agent, wrong for a retired one whose brain repo
  is archived and can no longer accept a push. Switching that timer off then made the *checker* nag
  nightly instead, trading one daily alarm for another. New **`DORMANT_AGENTS`** list inverts the
  assertion for those agents: the mirror must be **off**, so accidentally re-enabling it is caught
  too. Declarative on purpose — dormancy is a decision no code can infer from the filesystem, and the
  tempting alternative (reading a `DORMANT` banner out of the agent's own `CLAUDE.md`) would let
  untrusted repo content decide what the fleet checks.

Both branches were verified against deliberately-broken fixtures rather than a clean run, per this
file's own standing rule: a check that never fires looks exactly like a check that passes.

## [0.6.7] — 2026-08-07 — The docs catch up with the code

A fleet-wide audit of the reference deployment read the docs against the running system. The code was
fine; the prose had fallen behind it in ways that would mislead an adopter, and one template shipped a
violation of a rule the framework had just introduced.

### Fixed
- **`templates/kb-agent-template/OPERATING.md` shipped the duplication 0.6.6 abolished.** Its
  *Source of Truth & Bootstrap* section still carried a full numbered read order — with repo-relative
  paths, and a step 2 pointing at itself — so every newly instantiated agent started life in violation
  of the one-place rule. 0.6.6 fixed `agent-template.md` and `bootstrap.md` and missed this file, which
  is the one adopters actually copy. Now it states where the read order lives and why, and the
  frontmatter no longer claims to contain it.
- **`templates/infra/scripts/provision-agent`: restored `'\''` escaping** in the rotate-on-boot warning.
  The bare quotes closed the surrounding single-quoted block; they happened to re-balance, so the script
  ran — but a security-sensitive root-run region was left open to the outer shell's word splitting.
- **`templates/kb-agent-shared/runbooks/fleet-brain-change.md` contradicted itself**: the shared-repo
  paragraph still prescribed `runuser` + piping scripts to `python3 -`, while the procedure below it
  prescribed the opposite. It now uses one form throughout, and states the shared-write choice
  explicitly instead of assuming one.

### Documentation
- **`docs/runtime.md` — the whole rotate/bridge arc was missing.** The command list omitted `rotate`,
  `rotate-all`, `urls` and `remember`; nothing mentioned `claude-topic-rotate-on-boot.service`; and the
  narrative still promised the wrapper's job was to *never lose conversation history*, which 0.6.5
  deliberately traded away. New section **"`active` is not `reachable`"** states the failure mode
  plainly: the bridge is server-side state, no local probe can see it, `restart` re-announces the dead
  bridge, only `rotate` mints a new one — and the reboot cost (every topic starts fresh) is named
  rather than buried.
- **`docs/monitoring.md`** — hysteresis is no longer described as unconditional: memory exhaustion
  alarms on first detection (0.6.4), because the second cycle may never run on a box deep into swap.
  Adds the limit the monitor cannot cover: topics `active` ≠ bridges reachable.
- **`docs/reference-architecture.md`** — the fourth agent has been dormant since 2026-08-03; the table
  said it was active and gave its scope to the wrong agent. Now records **how** an agent is retired
  (archive the brain, keep the user and the roster entry, switch off its unattended writers) as a
  reusable lesson. Also corrects "no public inbound ports": that is a property of the services you put
  behind a tunnel, not of the box — the reference deployment's own control plane listened on `0.0.0.0`,
  and a host firewall would not have helped, because Docker publishes past it.
- **`docs/portability.md`** — `topics.rotated` and `last-boot-rotation.tsv` added to the not-carried table.
- **`templates/kb-agent-shared/runbooks/agent-ops-and-portability.md`** — same rotate/bridge gap as
  `runtime.md`, in the runbook agents actually load.
- **`templates/kb-agent-template/memory/README.md`** — documents the machine-owned `auto/` tier and
  that it must not be hand-edited, plus the asymmetry that bites: mirroring is automatic, distilling
  is not.

### Added
- **`agentic-divergence-check` reports a CHANGELOG head that no tag matches.** This release exists
  because the audit found three releases (0.6.4–0.6.6) that were documented, committed and pushed but
  never tagged — invisible to `git describe`, to the releases page, and to anyone pinning a tag. The
  tree was clean and in sync, which is exactly why nothing caught it. Now the daily drift report does.

## [0.6.6] — 2026-08-06 — One place per fact

The framework mandated its own duplication. `agent-template.md` required a *Bootstrap Contract*
section inside `OPERATING.md`, while the next bullet of the same file declared `CLAUDE.md` to be **the**
bootstrap — thin, absolute paths, not restating what it points at. Every brain faithfully implemented
both halves, so the read order existed three times per agent (`CLAUDE.md`, `OPERATING.md`,
`shared/bootstrap.md`) and drifted apart, as duplicated text does.

Two bugs were riding in the duplicates, in every brain: repo-relative paths, which do not resolve from
`cwd=~` — the trap the template itself warns about two lines below the mandate — and a step ordering a
read of `CLAUDE.md`, which the harness has already loaded before the agent reads anything. A no-op,
stated three times.

### Changed
- **`agent-template.md`: no bootstrap section in `OPERATING.md`.** The read order lives in `CLAUDE.md`,
  once, with absolute paths. Adds **"The one-place rule"** — *a fact belongs in the always-loaded layer
  only if the agent cannot know to go looking for it; everything else is retrieved on demand and stated
  exactly once* — and extends it below the read order: a hard rule written verbatim in `CLAUDE.md` and
  restated at the head of its `OPERATING.md` section is two copies of one rule.
- **`agent-template.md`: `CLAUDE.md` is THE bootstrap.** The template still described it as an optional
  "~15-line adapter" for non-Claude harnesses — the superseded two-file model, and incoherent with the
  change above. Now matches how `provision-agent` actually wires it (`~/CLAUDE.md` symlink, unit sets
  `WorkingDirectory=%h`), with the `AGENTS.md` sibling demoted to the optional adapter it is.
- **`bootstrap.md`: "Bootstrap Order" replaced by a pointer.** The only non-duplicated content in that
  section — on-demand reads, no whole directories — is kept.
- **`runbooks/fleet-brain-change.md`:** prefer `sudo -u` over `sudo runuser -u` (a permission classifier
  may pass the runuser form on read-only preflight, then block it on the first write; `--noprofile
  --norc` plus an explicit PATH is what actually carries the safety property). Editing guidance moves
  from `python3 -` pipes to a guarded `head`/`sed` rebuild **with boundary assertions**, because the
  brains are live and a section can be rewritten under you between read and write.
- **`runbooks/fleet-brain-change.md`: gate `git commit` on `git add` with `&&`.** Learned expensively:
  `git add` aborts on a pathspec matching nothing and stages nothing — the trigger being a path already
  moved by `git mv`. Ungated, the commit still runs, ships with zero insertions, gets pushed, and leaves
  the real changes uncommitted in a dirty tree, which is precisely the state that makes the sync job
  skip that repo silently and forever. Both halves invisible unless you look.

### Not a token optimization
Stated in the template so it is not misread later. Measured on the reference fleet, harness 2.1.223: a
fresh session loads 19.6k of 1.0M (2%), MCP schemas deferred. There was nothing meaningful to reclaim.
The cost of three copies is that they disagree and each one looks authoritative — one had already
drifted out of agreement with its own `CLAUDE.md`, and another carried a stale delegation map naming an
agent that had been dormant for three days.

## [0.6.5] — 2026-08-06 — Stop checking, start rotating

`0.6.4` gave operators the tools to recover from a dead Remote Control bridge — `rotate` to fix
one, `urls` to find which. It left the recovery itself manual: after every reboot a human had to
open twelve URLs and judge each one. That is a chore nobody will keep doing, which makes it a
fix with a shelf life.

The obvious automation is forbidden by `0.6.4`'s own finding. A dead bridge is **not observable
from the host** — every local signal records what the session *announced*, not what the server
honours — so "detect the broken ones and heal them" reports green for exactly the failure it
exists to catch. Any check built here is a liability.

The way out is to stop trying to observe. A new conversation always mints a live bridge, so
rotating *everything* at boot is correct by construction: no check, therefore no false green.

### Added
- **`claude-topic rotate-all [--dry-run]`** — rotate every *enabled* topic in one pass. Only
  enabled units are touched: `stop` disables deliberately, and a topic someone stopped on purpose
  must not be resurrected by a reboot. Each rotate runs in a subshell so one failure cannot
  abandon the remaining topics half-done, and the command writes
  `~/.config/agent/last-boot-rotation.tsv` — key, display name, abandoned sessionId, new URL —
  refusing to rotate at all if it cannot create that digest first.
- **`claude-topic-rotate-on-boot.service`** — a `oneshot` user unit, `WantedBy`/`After=default.target`,
  that runs `rotate-all` once per boot. `TimeoutStartSec=600`, stated explicitly because
  `Type=oneshot` inherits `TimeoutStartUSec=infinity` — the same default that wedged the monitor's
  timer through the outage in `0.6.4`.
- **`provision-agent` installs and enables it** — `enable`, deliberately **not** `--now`:
  running it during provisioning would rotate the live conversations of an agent being
  *re*-provisioned. The new unit joins the dirty-worktree guard alongside the wrapper.

### The trade, stated plainly
Every topic returns from a reboot with **empty context**. History is not lost: each abandoned
sessionId is appended to `topics.rotated` before anything is discarded, transcripts are never
deleted, and the per-boot digest says where each one went. Durable knowledge belongs in the
agent's brain repo and `memory/`, not in a topic's scroll-back — and a reboot that hurts is a
signal the knowledge was in the wrong place.

Failure is bounded by design: the topic units are independent and start regardless, so a broken
`rotate-all` leaves a fleet exactly where `0.6.4` left it — up, with bridges to test by hand.
It cannot prevent topics from coming back.

### Verified
The boot path was proven before shipping, not assumed: restarting a lingering user's manager —
what actually happens at boot — activated the unit and ran it to `Result=success`,
`ExecMainStatus=0`, with start/output/finish all captured in the journal. Tested on a dormant
agent so no live conversation was spent on the experiment; the dormant agent also confirmed the
enabled-only guard, reporting `no enabled topics to rotate` and changing nothing.

## [0.6.4] — 2026-08-06 — Three green checks and a total outage

A host ran out of memory and thrashed until it was rebooted. Every topic session came back
`active` and `enabled`; eleven of twelve were permanently unreachable from the app. Three
independent monitoring layers reported healthy throughout. The framework had no way to say
otherwise, and — worse — encouraged the belief that `Restart=always` covered this.

### Added
- **`claude-topic rotate <key>`** — the recovery verb for an unreachable topic. The Remote Control
  bridge id is derived from the *resumed conversation*, so a restart re-announces the same id;
  when the server has dropped that bridge, restarting is a no-op and the topic is unreachable
  forever. `rotate` abandons the conversation and starts a fresh one, appending the abandoned
  sessionId to `topics.rotated` **before** discarding anything, so nothing becomes unaddressable.
  It waits for the new bridge and prints the URL, because the only real confirmation is a human
  opening it.
- **`claude-topic urls`** — every topic's Remote Control URL in one command. Recovering from the
  incident meant hand-parsing `~/.claude/sessions/*.json` against `/proc`; this makes the only
  honest health check a five-second operation. `status` now shows the bridge URL too.

### Changed
- **`agentic-monitor`: `TimeoutStartSec=120`.** `Type=oneshot` inherits
  `TimeoutStartUSec=infinity`, so the run that started while the host was thrashing never
  returned — no alarm, no heartbeat, timer blocked. A wedged run now dies and surfaces both as a
  failed unit and as a missed beat on the dead-man's switch.
- **Memory alarms on first detection.** The 2-cycle hysteresis is correct for flapping checks and
  wrong for memory, because the second cycle is the one that wedges. Memory sets `urgent=1`;
  every other check keeps the hysteresis.
- **Memory threshold 5% → 15%**, in the script default and `agentic-monitor.env.example`. At 5%
  the host may no longer be able to send the ping. Note the deployed env file pins this key and
  overrides the script default — changing the script alone is inert.

### Documented
- **`Restart=always` does not keep a topic reachable**, stated in `claude-topic@.service`. It
  keeps the process alive. Those are different properties and the difference was an outage.
- **Do not build a local reachability check.** Every host-visible signal records what the session
  *announced*, not what the server honours, so such a check reports green for exactly the failure
  it claims to catch. The scope limit is written into both `bin/claude-topic` and the topics check
  in `agentic-monitor`, so the gap is not "fixed" by making it invisible.

## [0.6.3] — 2026-08-02 — Guards that ran after the mutation, and guards that opened when they could not tell

v0.6.2 moved `provision-agent`'s read-only checks above its first mutation and wrote down why: *a
check that can run first must run first; only the mutation belongs late*. A second audit — plus an
independent adversarial review briefed to refute it — found that the reasoning had never crossed the
file boundary, and that three guards were weaker than they read. Every fix below was proven against a
deliberately-broken fixture, and a legitimate install was re-run each time to prove it still succeeds.

### Security
- **`install-host-services` installed six root-owned artifacts before any guard could speak.**
  `kb-sync`, `agentic-monitor` and their four systemd units were written to `/usr/local/bin` and
  `/etc/systemd/system` at the top of the script; the git guards sat two thirds of the way down. A
  guard that fires there does not prevent a bad install — it **guarantees a half-finished one**.
  Reproduced; the reference deployment still carried the evidence (an orphaned mode-0600 manifest
  temp file holding exactly those six rows, from a run that died at the old guard). Every read-only
  precondition — worktree, unpushed commits, roster-shrink — now runs at step 0 in **both**
  installers, before anything is written.
- **The dirty-worktree guard only ever looked at `bin/claude-topic`.** Uncommitted changes to
  `kb-sync`, `agentic-monitor`, `agentic-divergence-check` and `memory-mirror` installed root-owned
  with no check at all — and `kb-sync` runs as root every fifteen minutes across every agent home.
  Reproduced: injected code in `scripts/kb-sync` installed, exit 0. The check now covers everything
  the installer installs, and uses `git status --porcelain` rather than `git diff`, which exits 0
  for a path git does not track — so an **untracked** `bin/claude-topic` used to sail through too.
- **The unpushed-commits guard was skipped whenever it could not answer.** `AHEAD=$(… || echo
  unknown)` followed by `[ "$AHEAD" != unknown ]` meant a clone with no upstream — a locally created
  branch, a detached HEAD, or an infra repo not yet pushed, which is exactly the fresh-host bootstrap
  this script is for — bypassed it entirely. Reproduced: an unreviewed, unpushed wrapper installed
  fleet-wide with the guard printing `[ok]`. A missing upstream is now itself a FAIL. The correct
  shape already existed three times in this codebase (`claude-topic`'s `reg_commit`, `memory-mirror`,
  `agentic-divergence-check`): ask about `@{u}` first, treat its absence as a finding.
- **Two shipped documents taught the bypass.** `claude-topic@.service` — a file copied into *every*
  agent's home — carried an "install, as that agent's user" comment beginning with `sudo install …
  /usr/local/bin/claude-topic`: an instruction telling an unprivileged agent to root-install the
  binary every other agent executes, from its own clone, with none of these guards. `monitoring.md`
  offered the same shortcut for `agentic-monitor`, which additionally writes no manifest row. Both
  removed; `agent-ops-and-portability.md`'s hand-run provisioning block is now one `provision-agent`
  call.

### Fixed
- **An emptied roster silently switched three assertion families off.** The per-brain loop iterated
  the union of roster and discovered agents; the mirror-timer, stale-timer and supporting-repo
  sections still iterated the roster alone. With an existing-but-empty roster the checker therefore
  stopped verifying `infra` and `kb-agent-shared` entirely — including the `STRICT_CLEAN_REPOS`
  dirty-worktree assertion that backstops both installers — while simultaneously reporting all four
  live `memory-mirror@` timers as `enabled but not in the roster (stale teardown?)`, a message whose
  remedy is to **disable durable memory for the whole fleet**. Reproduced: 8 findings, 4 of them
  wrong, and nothing indicating that anything had stopped being asserted. The set is now computed
  once and used everywhere. This is the class v0.6.1 called out and fixed one section higher up.
- **`memory-mirror` picked the brain repo with `head -1`.** It is an unattended writer that
  `rsync --delete`s into the repo it picks and then commits and pushes there, so with two candidates
  (a rename left half-done) it published an agent's memory into the **wrong repo** — while
  `claude-topic`, facing the identical ambiguity, refuses to start at all. The writer is now at least
  as cautious as the launcher; `agentic-divergence-check` refuses the same ambiguity instead of
  auditing whichever repo it happened to pick.
- **Nothing verified what the manifest does *not* contain.** The installed-artifact check iterates
  rows in the manifest, so it proves "everything installed is listed" and never the reverse — which
  is how a root-owned temp file sat beside the very file that loop reads, unseen. A sweep now reports
  anything in `/usr/local/share/agentic` the manifest does not account for. Deliberately not extended
  to `/usr/local/bin`, which is shared with the rest of the system: sweeping there would flag every
  binary a host legitimately has. `install-host-services` also now `trap`s its temp manifest, so the
  leftovers stop being created in the first place.
- **`INFRA_DIR` — the checker's own root of trust — was asserted to be nothing.** It is derived from
  the manifest, i.e. from whichever clone last ran the installer, and every "installed matches
  source" comparison, the `AGENT_PATH` assertion and the per-agent unit comparison resolve through
  it. A run from a throwaway clone silently re-points all of them, after which the comparison
  compares the odd install against the odd source and agrees. v0.6.1 closed the read side of this;
  the write side stayed open. The checker now asserts that its root of trust is a clone it actually
  audits, and `docs/divergence-check.md` no longer claims the manifest's coverage "cannot drift" —
  it states precisely which direction it proves.
- A roster name with no Unix user reported `no brain repo found under /github/<org>/` — a path that
  does not exist and does not name the real problem. Same dead-`||` idiom already fixed in `kb-sync`,
  `memory-mirror` and `provision-agent`; this was the fourth copy.

## [0.6.2] — 2026-08-01 — What an end-to-end provisioning test found (and what the review of those fixes found)

v0.6.1 was reviewed but never *executed*: nobody had run the documented bring-up on a real machine.
This release is the result of doing that — a throwaway Unix user, these templates, a stubbed runtime,
local git origins — plus an adversarial review of the fixes it produced. Nine defects, every one
reproduced live and re-proven fixed by re-running the same test.

### Security
- **`install-host-services` was a complete bypass of `provision-agent`'s protections.** It installed
  the root-owned fleet-wide `claude-topic` wrapper *and* the fleet roster with none of the
  dirty-worktree / unpushed / roster-shrink guards. On an established box (where "it's the fresh-host
  bootstrap" stops being true) that is the same root-execution channel the config model exists to
  forbid, reachable through the other door. Both guards now apply there too.
- **The fleet-wide wrapper install no longer runs first.** It was step 1, so a provisioning that
  failed at step 2 still left *every other agent's* wrapper replaced, with nothing to roll it back —
  reproduced when a failed test run swapped the live wrapper. It now runs only after this agent's own
  clones, symlinks and settings are in place; the read-only git guards that can abort it stay at
  step 0, before anything is mutated at all.

### Fixed
- **Provisioning silently redefined the whole fleet.** Refreshing the installed roster from the
  provisioner's clone replaced it wholesale: reproduced end-to-end, one run left
  `/usr/local/share/agentic/fleet-agents` containing **only** the newly-provisioned agent, so
  `kb-sync` stopped syncing and `agentic-monitor` stopped watching four live agents — silently, since
  a monitor cannot alarm about agents it no longer knows. Both installers now **refuse** a roster
  that drops names which still have a Unix user and a brain repo on the box, and the roster is only
  mutated after that guard can no longer abort. Verified it fires on that exact case and does not
  block a legitimate superset.
- **A freshly provisioned agent's topics had no stored sessionId at all** — `topics.state` did not
  even exist. `claude-topic list` showed `active` with an empty session, which the runbook itself
  calls unhealthy, and the next reboot would have started **blank conversations and orphaned the
  history** (the incident class the wrapper exists to prevent, reached through the provisioning
  door). Provisioning now captures each sessionId, with a retry budget larger than the 15s the
  wrapper itself allows for the session file to appear.
- **Nothing could detect that state afterwards.** Neither the monitor nor the divergence check ever
  looked at `topics.state`, so a scrolled-past warning was the only signal. The daily check now
  asserts that every **enabled** topic has a stored sessionId — fixture-proven to fire on a removed
  state row and to stay silent on a healthy fleet.
- **Roster comparisons now strip CR.** With CRLF line endings the new guard both false-positived
  (blocking a legitimate run) and false-negatived (waving a drop through). Fixed in every roster
  reader.
- **A brand-new brain was dirty from minute one**: the fleet-common skill symlinks provisioning
  creates were untracked, and nothing said so. Provisioning now reports them (it deliberately never
  commits into an agent's own repo) and the runbook says to commit them.
- **Every adopter's brain reported four broken references on day one.** The brain template's README
  pointed at `docs/*.md` files that exist in *this* repo, not in a deployed brain — four standing
  findings in the daily report, which is how a report teaches you to ignore it. The pointers are now
  exact and outside backticks, so they are precise without being checked as repo-relative paths.
- **The bring-up never installed `gh` or `rsync`**, which the documented procedure hard-requires: a
  literal stranger dead-ended at the token step with `gh: command not found`.

### Verified working (the point of running it)
The `gh` precondition blocks as designed; `memory-mirror` refused to push a repo that already had
unpushed work and exited non-zero; `remember` + `restart` resumed the same conversation; the
committed `shared` symlink survives `cp -r` and resolves; teardown via the REMOVE checklist left no
failed unit, no timer and no home behind.

## [0.6.1] — 2026-07-30 — What the adversarial review of v0.6.0 found

v0.6.0 shipped with "no known holes"; an independent adversarial review (a fresh agent, briefed to
refute, reading the repo as a stranger) found ten real issues the same afternoon — including one the
release itself introduced. Every claim was re-verified line-by-line before fixing; the assertions
below were fixture-proven to fire.

### Security
- **`provision-agent` sourced the fleet-wide root-owned wrapper from the first `infra` clone in ANY
  agent's home.** Introduced by v0.6.0's generalization (the reference deployment pins the path). Any
  unprivileged agent with org read could clone `infra` into its own home with malicious *committed*
  changes — the dirty-worktree guard passes on committed changes — and alphabetical first-match would
  install its `bin/claude-topic` root-owned for every agent. Now resolved from the **invoking user's**
  home (`SUDO_USER`), and a clone **ahead of origin is a hard FAIL** (unpushed commits must not reach
  a fleet-wide root install). `agentic-divergence-check` had the same first-match glob for its
  comparison source; it now derives `INFRA_DIR` from the root-written manifest instead.
- `claude-topic new` rejects newlines/CR in display names (a multi-line name wrote a second, valid
  row into the TSV registry — row injection).

### Fixed
- **`agentic-monitor` silently dropped the last registry line when it lacked a trailing newline**
  (`while read` returns 1 on an unterminated final line) — that topic was simply never monitored.
  Now read via charset-filtered awk, like every other registry consumer. Fixture-proven.
- **The dead-man's switch ran green against the placeholder URL.** The example env pre-filled
  `HC_URL=https://hc-ping.com/REPLACE-…`, the `:?` guard only catches *unset*, and ping failures are
  deliberately swallowed — skip the monitoring step and the monitor reports success forever while
  pinging nothing. The example now ships empty values (the guard fires loudly) and a still-a-
  placeholder URL is FATAL.
- **The roster reconciliation was a no-op in exactly the shipped state.** With an existing-but-empty
  roster (what the template ships), `AGENTS` fell back to the discovered set and the both-ways
  comparison compared discovery against itself. The fallback now applies only when the roster *file*
  is missing; an empty roster with brains on disk drifts per brain, and per-brain checks iterate the
  union. Fixture-proven (8 findings fire on a live fleet with an emptied roster).
- **The walkthrough never installs `gh` or `rsync`**, which the procedure hard-requires (token
  wiring, provisioning auth check, nightly mirror): a literal stranger dead-ended at step 7 with
  `gh: command not found`. Both docs now install the four runtime deps explicitly.
- **The shipped permission template allowlisted the `topic` wrapper this release deleted** (and not
  `claude-topic`, which agents actually run) — every newly-provisioned agent got dead allowlist
  entries. Fixed, along with the wrong `node` path.
- **`docs/memory.md` said the mirror "ships disabled — enable when you want it"**, contradicting
  v0.6.0's own provision-enables-it-with-disclosure behavior *and* the divergence check that asserts
  the timer on. The disclosure a cautious adopter reads first now matches what happens.
- `manage-agents.md` used paths that don't exist in a deployed org (`templates/infra/...`), taught
  hand-editing the roster that `provision-agent` now maintains, and its REMOVE checklist omitted the
  memory-mirror timer (a leftover instance fails nightly after `userdel` and pages every cycle) and
  the roster line. All three fixed.
- Stale text: `agent-ops-and-portability.md` still said the bare `topic` symlink survives; the env
  example still named the deleted weekly `agentic-update-check`; the divergence timer's comment still
  told a human to enable what the installer enables; `README_AGENT` claimed a misnamed brain makes
  `kb-sync` skip it (kb-sync is roster-driven and name-agnostic — it is the monitor, the divergence
  check and the mirror that go blind). The brain template's `.gitignore` carried a duplicate
  `hosts.yml` and a `deploy/topics.state` path that never existed.

## [0.6.0] — 2026-07-30 — The audit release: propagate everything the reference deployment learned, delete what drifted

A full adversarial audit of both sides (the reference deployment and this repo) found that a dozen
safety/correctness fixes had landed privately without their `[Unreleased]` line ever being written —
so this release ports **all** of them, replaces that convention with a mechanical release-time diff
(see CONTRIBUTING §Releases), and deletes the pieces whose absence was an improvement. Every
assertion added here was proven against a deliberately-broken fixture before shipping; a CLEAN run
was never accepted as evidence.

### Security
- **`bash -lc` eliminated from every script that runs commands as an agent** (`agentic-divergence-check`,
  `memory-mirror`, `provision-agent` — `agentic-monitor` was fixed in 0.4.1). A login shell sources the
  *audited agent's own* `~/.profile`, so one planted shell function made the drift detector report CLEAN
  because the thing it audits told it to. Now `bash --noprofile --norc -c` with an explicit `PATH` that
  deliberately excludes agent-writable dirs; the four identical `AGENT_PATH` definitions are themselves
  asserted identical by the daily check. The `fleet-brain-change` runbook's copy-paste examples taught
  the same hole for interactive cross-agent maintenance — hardened the same way.
- **Registry keys are untrusted input to root's shell.** The divergence check reads `deploy/topics.tsv`
  (agent-writable); keys are now charset-filtered and passed as positional arguments, never interpolated —
  a crafted key could otherwise suppress its own drift finding.

### Fixed
- **`memory-mirror` committed and pushed work that was not its own.** Unscoped `git add`/`diff`/`commit`
  swept an agent's staged half-finished edits into the nightly `[mirror]` commit, and an unconditional
  `git push` published every pre-existing unpushed commit, unreviewed, on the timer's clock. All git
  operations are now path-scoped to `memory/auto`, the pre-existing `ahead` count is read *before*
  committing, and the job refuses to push anything but its own work.
- **`provision-agent` could delete an agent's settings.** `rm -f settings.json && install` under
  `set -e` meant a failed install exited with the permission allowlist *gone*. Now installs to a temp
  name and `mv`s atomically. Also: the topic registry is parsed on literal TABs (a `read key name` split
  on any whitespace and silently dropped a final line with no trailing newline); installing the fleet-wide
  wrapper from a dirty or behind-origin infra clone is refused/warned instead of silent; a broken
  provisioning now exits non-zero for the memory-mirror step too.
- **`claude-topic` mutations of the versioned registry are now durable** (`reg_commit`): registering or
  removing a topic commits and pushes `deploy/topics.tsv` path-scoped — with a checked `add` for the
  brand-new-registry case and a refuse-to-push guard when the repo already has unpushed commits (pushing
  would publish unrelated work and mask the divergence the daily check reports). A failed `new` now rolls
  back all three of its half-built artifacts (registry row, enabled unit, stored sessionId) instead of
  leaving orphans no check could see.
- **`agentic-divergence-check` had fail-opens and blind spots**, all fixture-proven closed: a branch with
  no upstream read as "in sync" (both rev-list counts silently 0); a failed fetch read as clean; a missing
  roster was a stderr warning nothing reads. New assertions: `topics.tsv` tracked + matching HEAD +
  agreeing both directions with the systemd units actually enabled; per-agent installed artifacts
  (`claude-settings.json`, the topic unit, and **any** `deploy/*.service|*.timer`) match their sources —
  and, in reverse, **no enabled user unit exists that the brain does not declare** (found live: an MCP
  service existing only on the box, secret inline, that a rebuild would have silently lost); linger per
  agent; a `memory-mirror@` timer enabled per roster agent — enumerated via `list-units`, because
  `list-unit-files --state=enabled` returns nothing for template instances and the first version of that
  reverse check was dead code that could never fire.
- **`kb-sync` exited 0 on a 100% skip rate** — a dead token or GitHub outage was indistinguishable from a
  clean run. All-repos-failed now exits 1 (the monitor's 2-cycle hysteresis absorbs transients); a single
  skip stays non-fatal (an agent's WIP is normal).
- **`install-host-services` left its own jobs disabled** and printed "[ok] timers enabled" even when
  enabling failed. It now enables everything it ships (a checker that is off is worthless, and "enable it
  later" is a step that does not happen), reports per-timer failures honestly, installs the `claude-topic`
  wrapper (a fresh host previously got a manifest row pointing at a file the installer never wrote), and
  derives `installed.manifest` from the installs themselves — the hand-maintained pair list (add an
  install, forget the pair, artifact unverified forever) cannot drift anymore. Per-agent memory-mirror
  timers stay gated behind `--enable-writers`/provisioning, with disclosure: they are unattended writers.
- **`agentic-monitor` now sweeps failed per-agent *user* units** — an agent's own service (e.g. an MCP
  server) could fail forever with no alarm — and reads the fleet roster as its source of truth, alarming
  (while falling back to on-disk discovery) if the roster is missing rather than silently narrowing its
  own scope.
- **`claude-topic@.service` could park every topic in `failed` at boot.** `After=network-online.target`
  is a no-op in the user manager (the target does not exist there), so topics start before the network;
  with `StartLimitBurst=5`, five fast failures wedged the unit until a human ran `reset-failed` — on a
  box whose point is unattended recovery. The start limit is gone (a permanently-broken topic flaps
  loudly instead, which the monitor pages on), `RestartSec` is 10s, and the misleading dependency is
  removed with a comment explaining why.
- **The `shared` symlink the architecture describes was never actually committed** by the documented
  bring-up (it was created *after* the seed commit and never pushed). The brain template now ships it.
- The daily check's report line and the divergence-check/monitoring/patch-management docs described the
  superseded weekly-report arrangement; all rewritten for the folded daily model.

### Changed
- **One bootstrap layout, deliberately.** `deploy/home-CLAUDE.md` and `AGENTS.md` are **removed**; the
  brain's root `CLAUDE.md` (absolute paths, symlinked to `~/CLAUDE.md`) is the single entry point, and
  `provision-agent`/`agentic-divergence-check` accept only it. The wiring is honestly Claude-Code-only —
  Remote Control is the reason this stack exists; the brain *content* remains portable Markdown
  (`docs/portability.md` rewritten accordingly).
- **The weekly `agentic-update-check` is folded into the daily divergence check** (deleted: its script and
  units). One file checks and alarms only when there is something to act on, the drift findings ride in
  the alert body itself, and the timer runs at 08:00 — after `apt-daily-upgrade`'s window, so a security
  update about to be auto-installed is not reported as pending.
- **The fleet roster is authoritative everywhere** (`kb-sync` refuses to run without it, the monitor
  alarms on its absence, the divergence check reconciles it against disk both ways), and it is
  **maintained by `provision-agent`**, not by hand — the template ships it empty.
- `approval-policy.md`: commit **and push** are one atomic unit of work — an unpushed commit is not a
  finished task (learned when push silently became optional busywork and clones drifted).
- `agent-audit`: new step 0 — refresh `shared/` first, or a freshly-edited fleet-common skill executes
  with its previous body; findings are now gathered into one batched decision instead of five
  interruptions.
- `fleet-brain-change`: a dirty `deploy/topics.tsv` is no longer "normal runtime dirt" to keep out of
  your commit — after `reg_commit` it is the signature of a failed registry commit, and the runbook now
  says to investigate it.
- `CONTRIBUTING.md` §Releases: the release delta is **derived by diffing `templates/` against the
  reference implementation**, hunk by hunk, instead of remembered via `[Unreleased]` entries written "at
  judgement time" — this audit proved that convention does not fire.
- `docs/reference-architecture.md`: the fleet is four agents (a finance/admin agent joined 2026-07-29),
  and the secrets section now describes the *actual* state (per-user token files restored by hand;
  Vaultwarden as the recommended store is the documented open item, not the achieved state).
- `docs/monitoring.md`: documents the user-unit sweep and makes `MONITOR_EXPECT_CONTAINERS` maintenance
  an owned step of your deploy/remove procedure, in both directions.

### Removed
- `templates/infra/scripts/agentic-update-check` + its service/timer (folded, above).
- `templates/kb-agent-template/deploy/home-CLAUDE.md` and `templates/kb-agent-template/AGENTS.md`
  (single-file bootstrap, above).
- The `[Unreleased]`-at-judgement-time working convention (replaced by the release-time diff).

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

[0.8.7]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.7
[0.8.6]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.6
[0.8.5]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.5
[0.8.4]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.4
[0.8.3]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.3
[0.8.2]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.2
[0.8.1]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.1
[0.8.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.8.0
[0.7.2]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.7.2
[0.7.1]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.7.1
[0.7.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.7.0
[0.6.8]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.8
[0.6.7]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.7
[0.6.6]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.6
[0.6.5]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.5
[0.6.4]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.4
[0.6.3]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.3
[0.6.2]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.2
[0.6.1]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.1
[0.6.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.6.0
[0.5.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.5.0
[0.4.1]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.4.1
[0.4.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.4.0
[0.3.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.3.0
[0.2.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.2.0
[0.1.0]: https://github.com/Valiant-Codex/agentic-codex/releases/tag/v0.1.0
