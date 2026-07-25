<!-- title: Docs -->
# Docs — how and why it works

**New to servers?** Start with **[getting-started.md](getting-started.md)** — a click-by-click VPS +
Tailscale setup walkthrough for the human prerequisites.

The write-up behind the templates. Read in this order for the full picture:

1. **[architecture.md](architecture.md)** — the three layers (brains, governance, host) and how they fit.
2. **[config-model.md](config-model.md)** — the three-tier boundary (git / kb-sync / provision) that makes it portable *and* safe. The key idea.
3. **[portability.md](portability.md)** — canonical `SOUL.md + OPERATING.md` + thin `CLAUDE.md`/`AGENTS.md` adapters; recover/migrate in minutes.
4. **[runtime.md](runtime.md)** — topics, systemd supervision, the `claude-topic` wrapper, multi-device Remote Control.
5. **[skills.md](skills.md)** — folder-per-skill (`SKILL.md`), auto-registration, fleet-common skills, `skillify` + `agent-audit`.
6. **[memory.md](memory.md)** — two-tier memory, the nightly "dreaming" consolidation, and the human-gated autonomy line.
7. **[monitoring.md](monitoring.md)** — the dead-man's-switch: silence is the alarm.
8. **[secrets.md](secrets.md)** — secrets out of Git, in a store you can restore (Vaultwarden on Dokploy).
9. **[multi-agent-governance.md](multi-agent-governance.md)** — lanes, least privilege, and the honest limits of the separation.
10. **[app-layer.md](app-layer.md)** — the service layer (Dokploy / Cloudflare / apps): documented, not templated.
11. **[reference-architecture.md](reference-architecture.md)** — the real deployment this was distilled from.

To actually build it, follow **[../README_AGENT.md](../README_AGENT.md)**.
