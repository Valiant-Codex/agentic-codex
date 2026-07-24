<!-- title: Contributing -->
# Contributing

Thanks for looking under the hood. Agentic Codex is a **blueprint + templates**, so contributions are
mostly docs, templates, and portability fixes rather than a running app.

## Ways to help

- **Report a gap or a broken step** — open an issue describing what you were doing (following
  `README_AGENT.md`? adapting a template?) and where it snagged.
- **Improve the docs** — clarity fixes, missing prerequisites, better diagrams.
- **Harden the templates** — safer/cleaner scripts, more portable systemd units, another framework
  adapter alongside `CLAUDE.md` / `AGENTS.md`.
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

## Origin

This project was distilled and sanitized from the production system that runs
[Valiant Codex](https://github.com/Valiant-Codex)'s agents — see `docs/reference-architecture.md`.
MIT licensed; make it your own.
