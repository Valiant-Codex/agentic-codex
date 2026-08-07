# Changelog

All notable changes to **agentic-codex** are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/), and the project follows
[Semantic Versioning](https://semver.org/). Pre-1.0: the framework is still evolving, so minor
versions may include structural changes. `1.0.0` is reserved for a deliberate "stable and proven"
milestone.

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
