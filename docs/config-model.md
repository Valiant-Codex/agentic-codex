<!-- title: The Config Model — portability and security in one boundary -->
# The Config Model

This is the idea that makes the whole system portable *and* safe at the same time. Read it before you
change how anything is wired.

## The problem it solves

You want agents to be **portable**: move to another VPS, recover a broken one, or switch agent
framework, and get every brain and its configuration back with minimal effort — *clone and run*, not
*rebuild from scratch*. The obvious way to get there is to put everything in Git and auto-sync it into
the live locations on a short timer.

The trap: if one low-trust, unreviewed, auto-applied channel (git → 15-min sync → live location)
carries **everything**, it ends up carrying three payloads with very different trust levels:

| Payload | Trust | Safe to auto-apply? |
|---|---|---|
| Inert data (governance markdown, brain content) | low-risk | ✅ yes |
| **Executables** (the session wrapper, scripts) | high-risk | ❌ no |
| **Permission grants** (tool allowlists) | high-risk | ❌ no |

Auto-applying the last two turns your shared repo into a **root-execution channel** and your committed
allowlist into a **silent permission grant** — a hostile or mistaken commit reaches a live, executing
state with no review. Portability and security *look* opposed only because one mechanism is doing all
three jobs. Separate them and the tension disappears.

## The three tiers

### Tier 1 — Git is the portable source of truth
*Everything* portable lives in Git: agent brains (identity, memory, skills, tools), shared governance,
the `claude-topic` wrapper and systemd unit (in `infra`), a **curated** per-agent settings/allowlist
(`deploy/claude-settings.json` in each brain), and the provisioning script itself. Moving VPS loses
nothing. Secrets are the **only** exception — they never go in Git (see [`secrets.md`](secrets.md)).

### Tier 2 — `kb-sync` auto-refresh (every 15 min, no review)
A timer fast-forwards each repo. It refreshes **only inert data**. It must **never** be the thing that
installs an executable or grants a permission. Low-trust channel ⇒ low-risk payload only. Fail-safe: on
a non-fast-forward or dirty repo it logs `skip` and moves on — never forces, never merges.

### Tier 3 — `provision-agent` (explicit, run by the privileged agent, reviewable)
The **only** step that installs executables (to root-owned `/usr/local/bin`), writes systemd units,
materializes live settings as **real installed copies** (not live symlinks into the auto-synced repo),
enables linger, and starts sessions. It expects secrets to have been restored out-of-band and fails
fast if they're missing.

It is simultaneously:
- the **portability mechanism** — a new box is `provision-agent <user> <brain>`;
- the **security boundary** — a hostile or mistaken git commit cannot reach a live/executing state
  without a deliberate re-provision.

```
   Tier 1: GIT  ──────────────►  Tier 2: kb-sync (15m)  ──►  live inert data (brains, governance)
   (portable truth)     │                                     [safe to auto-apply]
                        │
                        └──────►  Tier 3: provision-agent  ──►  executables, systemd units, settings
                                  (explicit, reviewed)          [the review point + security boundary]
```

## What this buys you

- **The auto-loop's payload matches its trust.** `kb-sync` stays the dumb, frequent, unreviewed
  refresher it is — but only for data that's safe to apply blindly.
- **Portability is fully preserved.** Nothing leaves Git; the curated settings and the provisioning
  logic are versioned. A new box is reproducible from the repos + the restored secret.
- **Uses Claude Code's own two-file split correctly.** `deploy/claude-settings.json` (curated,
  intentional, portable) is installed as `~/.claude/settings.json`; `~/.claude/settings.local.json`
  (auto-accumulated per-session approvals — host-specific, security-sensitive noise) stays local,
  git-ignored, and is **never** carried.

## Consequences you should expect

- **Live settings changes no longer auto-propagate.** Changing a permission means editing
  `deploy/claude-settings.json` and re-provisioning, not a 15-minute pull. That's the intended review
  point, not a regression.
- **The wrapper is root-owned and only written by Tier-3.** The one thing every agent *executes* is
  installed by root and never touched by the auto-sync.
- **Moving VPS** = restore the agent's secret (its `gh` token), then `provision-agent <user> <brain>`.
  **Changing framework** requires rewriting the Tier-3 runtime wiring — the session wrapper, the
  systemd unit, the settings file and skills discovery are all runtime-specific. What moves unchanged is
  the valuable part: brains and governance, as framework-agnostic Markdown.

## What lives where (quick reference)

| Concern | Versioned in Git | Applied to the box by |
|---|---|---|
| Identity / memory / skills / tools | brain repo | kb-sync (inert) |
| Shared governance | `kb-agent-shared` | kb-sync (inert) |
| Curated settings / allowlist | `deploy/claude-settings.json` | **provision-agent** (real copy) |
| Session wrapper + systemd unit | `infra` | **provision-agent** (root-owned) |
| Secrets (tokens/keys) | **never** | restored out-of-band ([`secrets.md`](secrets.md)) |
| Per-session auto-approvals, session IDs, account state | **never** | volatile, stays on the box |
