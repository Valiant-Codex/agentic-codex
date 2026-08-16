<!-- title: Docs -->
# Docs — how and why it works

**New to servers?** Start with **[getting-started.md](getting-started.md)** — a click-by-click VPS +
Tailscale setup walkthrough for the human prerequisites.

The write-up behind the templates. Read in this order for the full picture:

1. **[architecture.md](architecture.md)** — the three layers (brains, governance, host) and how they fit.
2. **[config-model.md](config-model.md)** — the three-tier boundary (git / kb-sync / provision) that makes it portable *and* safe. The key idea.
3. **[context-budget.md](context-budget.md)** — what a session actually loads, and why your prose is not the expensive part.
4. **[portability.md](portability.md)** — one canonical `CLAUDE.md` per agent; recover/migrate in minutes.
5. **[runtime.md](runtime.md)** — topics, systemd supervision, the `claude-topic` wrapper, multi-device Remote Control.
6. **[skills.md](skills.md)** — folder-per-skill (`SKILL.md`), auto-registration, fleet-common skills, `skillify` + `agent-audit`.
7. **[memory.md](memory.md)** — two-tier memory, the human-gated autonomy line, and a published negative result.
8. **[divergence-check.md](divergence-check.md)** — the structural linter that keeps brains honest: invariants, not vocabulary.
9. **[monitoring.md](monitoring.md)** — the dead-man's-switch: silence is the alarm.
10. **[secrets.md](secrets.md)** — secrets out of Git, in a store you can restore (Vaultwarden on Dokploy).
11. **[multi-agent-governance.md](multi-agent-governance.md)** — lanes, least privilege, and the honest limits of the separation.
12. **[app-layer.md](app-layer.md)** — the service layer (Dokploy / Cloudflare / apps): documented, not templated.
13. **[reference-architecture.md](reference-architecture.md)** — the real deployment this was distilled from.

To actually build it, follow **[../README_AGENT.md](../README_AGENT.md)**.
