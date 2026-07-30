<!-- title: Contributing -->
# Contributing

Thanks for looking under the hood. Agentic Codex is a **blueprint + templates**, so contributions are
mostly docs, templates, and portability fixes rather than a running app.

## Ways to help

- **Report a gap or a broken step** — open an issue describing what you were doing (following
  `README_AGENT.md`? adapting a template?) and where it snagged.
- **Improve the docs** — clarity fixes, missing prerequisites, better diagrams.
- **Harden the templates** — safer/cleaner scripts, more portable systemd units.
- **Port the happy path** — notes or adapters for other agent runtimes or other Linux distros.

## Ground rules

1. **Never commit a secret.** No tokens, keys, real hostnames, capability URLs (healthchecks ping URLs
   included), emails, or client data — in code, examples, or history. Examples use placeholders
   (`<ORG>`, `<AGENT>`, `<ROLE>`, `<VPS_HOST>`, `<TZ>`, `<HC_URL>`, `<OWNER>`) or obviously-fake values
   (`acme-labs`, `acme-ops-01`). The `.gitignore` files exist to protect you — don't loosen them.
2. **Templates stay generic.** Anything under `templates/` must be reusable by anyone: no brand names,
   no environment specifics. The named worked example lives only in `docs/reference-architecture.md`.
3. **Keep the security model intact.** The three-tier boundary (git / kb-sync / provision) in
   `docs/config-model.md` is load-bearing — changes that let the 15-minute auto-sync install
   executables or grant permissions defeat the point. Explain the reasoning in your PR.
4. **Match the voice.** Lean, direct, honest about limitations (see how the docs treat "lanes are not a
   sandbox"). Update the relevant folder `README.md` when you add or rename files.

## PRs

Small, focused PRs with a clear description are easiest to review. If you're changing the security or
config model, say so explicitly and describe the trade-off.

## Releases

This repo is a **starting example**, not a mirror of the deployment it came from — divergence is the
expected steady state. Two things flow upstream: **framework-level** change (the memory model, a new
mechanism, a new fleet-common skill, the brain shape, the config model) and **every safety or
correctness fix**, without exception. Shipping strangers root bash with a bug we already fixed for
ourselves is a liability, not a personalisation.

**The delta is derived, not remembered.** An earlier convention asked for an `[Unreleased]` CHANGELOG
line to be written the moment a change was judged framework-level; in practice that step did not fire
(a full audit found a dozen unrecorded propagation-mandatory fixes). So the release process derives
the delta mechanically instead:

1. **Diff `templates/` against the reference implementation** (the private `infra/` and
   `kb-agent-shared/` it was distilled from) and account for every hunk: port it (generalized,
   placeholders per ground rule 2), or consciously leave it private (deployment-specific). The diff
   cannot forget; a convention can.
2. Write the CHANGELOG section for the release **from that reconciled diff**:
   `## [X.Y.Z] — YYYY-MM-DD — <headline>`. Pre-1.0, a batch of correctness/honesty fixes is a
   **minor**; a single scoped security/robustness fix is a **patch**.
3. Add the link definition at the bottom: `[X.Y.Z]: .../releases/tag/vX.Y.Z`.
4. Commit as `vX.Y.Z: <headline>`, then an **annotated** tag `vX.Y.Z` with subject
   `agentic-codex vX.Y.Z — <headline>`.
5. Push commits **and** tags (`git push && git push --tags`), then publish the GitHub Release from the
   tag so the CHANGELOG's links resolve.

Before tagging, sanity-check the mechanical things that have actually been missed before: every version
has a link definition, every shipped script still passes `bash -n`, and no
`Valiant‑Codex`-specific name survives under `templates/` (grep for the reference deployment's
agent names and org).

## Origin

This project was distilled and sanitized from the production system that runs
[Valiant Codex](https://github.com/Valiant-Codex)'s agents — see `docs/reference-architecture.md`.
MIT licensed; make it your own.
