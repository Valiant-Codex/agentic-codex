<!-- title: Docs -->
# Docs — how and why it works

The write-up behind the templates. Read in this order for the full picture:

1. **[architecture.md](architecture.md)** — the three layers (brains, governance, host) and how they fit.
2. **[config-model.md](config-model.md)** — the three-tier boundary (git / kb-sync / provision) that makes it portable *and* safe. The key idea.
3. **[portability.md](portability.md)** — canonical `system-prompt.md` + thin `CLAUDE.md`/`AGENTS.md` adapters; recover/migrate in minutes.
4. **[runtime.md](runtime.md)** — topics, systemd supervision, the `claude-topic` wrapper, multi-device Remote Control.
5. **[monitoring.md](monitoring.md)** — the dead-man's-switch: silence is the alarm.
6. **[secrets.md](secrets.md)** — secrets out of Git, in a store you can restore (Vaultwarden on Dokploy).
7. **[multi-agent-governance.md](multi-agent-governance.md)** — lanes, least privilege, and the honest limits of the separation.
8. **[app-layer.md](app-layer.md)** — the service layer (Dokploy / Cloudflare / apps): documented, not templated.
9. **[reference-architecture.md](reference-architecture.md)** — the real deployment this was distilled from.

To actually build it, follow **[../README_AGENT.md](../README_AGENT.md)**.
