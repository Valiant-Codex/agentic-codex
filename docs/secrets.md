<!-- title: Secrets — out of Git, in a store you can restore -->
# Secrets

The one rule that never bends: **secrets never go in Git.** Not in a brain repo, not in
`kb-agent-shared`, not in `infra` — nowhere. Everything else about an agent is versioned and portable;
secrets are the deliberate exception, and this doc is how you handle them without losing portability.

## What counts as a secret

- Each agent's **GitHub token** (its bot PAT), wired at `~/.config/gh/hosts.yml` (mode 600).
- **MCP server secrets** (API keys) — referenced by `${ENV}` placeholders in `.mcp.json`, with the real
  values in `~/.claude.json` (mode 600) and/or a `~/.config/<agent>/secrets.env` sourced by the unit.
- The monitoring **`HC_URL`** (a capability URL) in `/etc/agentic-monitor.env` (mode 600).
- Any provider tokens (VPS API, Cloudflare, deploy tokens, etc.).

If any secret passes through an agent's session and gets persisted anywhere in Git, treat it as exposed
and **rotate it**.

## How they're referenced (so the structure stays in Git, the value doesn't)

`.mcp.json` and unit files carry **placeholders**, never values:

```jsonc
// .mcp.json — structure is versioned, secrets are ${ENV}
{ "mcpServers": { "example": { "command": "npx", "args": ["-y", "some-mcp"],
  "env": { "SOME_API_KEY": "${SOME_API_KEY}" } } } }
```

The real `SOME_API_KEY` lives only on the box. This is what lets the config be public/portable while the
secret stays private.

## The recommended secret store: Vaultwarden on Dokploy

For a fresh-box restore you need the secrets somewhere **restorable without pasting each one by hand**.
The lean, self-hosted answer that fits this stack:

**Run [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (a lightweight Bitwarden-compatible
server) as a Dokploy app on your VPS**, and keep every agent token/key/capability-URL in it.

Why this fits:
- **Self-hosted, you own it** — same philosophy as owning your agents' brains; no third-party secret
  custodian.
- **One place to restore from** — a new/replacement box becomes: bring up Vaultwarden (or reach your
  existing instance), read the secrets back, wire them, then `provision-agent`. No secret ever touched
  Git.
- **Standard clients** — the owner reaches it from the Bitwarden apps/browser extension on any device.
- **Already in the stack** — it's just another Dokploy app alongside the rest (see
  [`app-layer.md`](app-layer.md)).

> ⚠️ **Back up Vaultwarden off the box, regularly.** Vaultwarden holds the very secrets a VPS rebuild
> needs — so if it lives *only* on the VPS and the box is lost (disk failure, provider incident,
> accidental wipe), you lose the keys to your own recovery. Treat its data as critical:
> - schedule a **regular, automated backup** of the Vaultwarden data (its SQLite DB / `data/` volume, or
>   a Bitwarden-format encrypted vault export) to storage **outside the VPS** — object storage, another
>   host, or the owner's machine;
> - keep Vaultwarden's **admin token** and the **backup's decryption passphrase** somewhere restorable
>   *before* the box exists (a password-manager entry the owner holds) — they're the one thing you can't
>   fetch from the store you're trying to restore;
> - **test a restore** occasionally — an untested backup is a guess, not a safety net.

## The restore flow (what "portable secrets" actually means)

```
new/replacement VPS
   ├─ 1. reach your secret store (Vaultwarden)               ← the only pre-box step
   ├─ 2. create Unix user(s); install Claude Code (+ Node)
   ├─ 3. wire each agent's gh token + MCP secrets from the store   (out-of-band)
   └─ 4. provision-agent <user> <brain>                       ← everything else from Git
```

## Guardrails

- `.gitignore` in every template already excludes `*.env`, `*.key`, `*.crt`, `hosts.yml`,
  `.claude.json`, and `settings.local.json`. Don't remove those lines.
- The privileged agent can update any secret on the owner's behalf from chat — the owner doesn't need
  shell access — but **creating, storing, or rotating a secret is a human-confirm gate** (see
  [`../templates/kb-agent-shared/policies/approval-policy.md`](../templates/kb-agent-shared/policies/approval-policy.md)).
- Never commit a real capability URL (healthchecks ping URLs included) — they're secrets too.
