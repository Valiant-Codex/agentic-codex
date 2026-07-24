<!-- title: The application layer (documented, not templated) -->
# The application layer

Agentic Codex templates the **agent + host layer**. The **application layer** — the services your
agents operate (a website, automations, databases, a secret store) — is **documented here, not shipped
as templates**, because it's the part that varies most between users. This doc shows the shape that
works well with the rest of the system, and how the reference deployment uses it.

## The shape

```
                Cloudflare (DNS + Tunnel + Access)
                          │  (no public inbound ports)
                          ▼
        ┌─────────────────────────────────────────────┐
        │  VPS: Dokploy  →  Traefik  →  app containers  │
        │   website · n8n · Vaultwarden · databases · … │
        └─────────────────────────────────────────────┘
                 ▲ owns the host          ▲ deploys apps
            root-agent (sudo)         dev-agent (scoped token)
```

- **[Dokploy](https://dokploy.com)** is the deployment layer on the VPS — a self-hosted PaaS over
  Docker (Traefik ingress, databases, backups, app management). It gives your agents a clean,
  scriptable surface (a CLI/MCP/API) instead of hand-rolled `docker run`.
- **[Cloudflare](https://www.cloudflare.com) Tunnel + Access** puts services online **without opening
  inbound ports** on the VPS: the tunnel dials out, and Access gates who can reach admin surfaces. DNS
  lives in the same place.
- **Apps** are whatever you run: a website, an automation engine (e.g. [n8n](https://n8n.io)), your
  **Vaultwarden** secret store ([`secrets.md`](secrets.md)), databases, etc.

## Who owns what (maps onto the lanes)

- The **root-agent** owns the *host* layer: it runs and updates Dokploy, holds the Cloudflare
  tunnel/DNS/Access config, manages backups, and does OS/Docker maintenance. It never hands root to
  anyone.
- A **dev-agent** owns the *app* layer: it builds and deploys the website/apps/automations using its
  **own scoped deploy token** (e.g. a Dokploy or Git deploy token) — it never has root and never touches
  Cloudflare/backups. The boundary between root-agent and dev-agent is exactly the dev-agent's token
  scope.

This keeps the blast radius small: deploying an app doesn't require, and can't reach, host privileges.

## How the reference deployment uses it

In the reference deployment (see [`reference-architecture.md`](reference-architecture.md)):

- the **root-agent (Durin)** runs Dokploy on the VPS, owns the Cloudflare zone (Tunnel + Access in front
  of admin UIs like Dokploy and a Cockpit console), runs the daily backups, and operates the
  `agentic-monitor` heartbeat;
- the **dev-agent (Celebrimbor)** deploys the marketing website and builds n8n automation workflows
  with its own deploy rights — routed through *its* token, not through root;
- **Vaultwarden** runs as a Dokploy app and holds every agent's secret, so a box rebuild restores
  secrets from one place — and is itself **backed up off the box, regularly** (a VPS incident would
  otherwise take the secret store down with everything else; see [`secrets.md`](secrets.md)).

## Why this is docs-only

Dokploy/Cloudflare/app configs are specific to your domains, providers, and services, and they change
often. Templating them would be brittle and would tempt you to commit environment specifics. The stable,
reusable parts — the *agent + host* layer and the *ownership boundary* above — are what this repo ships.
Wire the app layer to taste; keep its secrets in your store, not in Git.
