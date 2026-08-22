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

## Build discipline for the dev-agent

The dev-agent is the one agent in a fleet that writes application code, so it is the one whose contract
carries a **build-discipline ladder** — YAGNI, reuse before rewrite, stdlib and native platform features
before dependencies, one line before fifty (the `Build discipline` section of
`templates/kb-agent-template/CLAUDE.md`). Coding agents over-build by default, and the ladder is the
cheapest correction available: a page of prose in the file the runtime already loads.

**Copy the ladder as prose; do not install a plugin for it.** The idea is
[ponytail](https://github.com/DietrichGebert/ponytail) (MIT), which packages the same ladder for 20+
agent hosts. Three reasons the packaged form does not fit an agent fleet:

- **It executes code in every session.** Its hooks run `node` at `SessionStart`, `SubagentStart` and
  `UserPromptSubmit`, with the agent's own privileges, updated through a plugin marketplace nobody on
  your side reviews. On a box where one agent holds sudo, that is a standing code-execution channel into
  the privileged context — the exact surface this governance model exists to close.
- **It biases your reviewers.** The `SubagentStart` hook ships with no matcher, so "at most three short
  lines, no essays" reaches code-review, audit and security subagents too. A reviewer told to be brief
  reports fewer findings.
- **The token saving is not the point, and is smaller than advertised.** The injected ruleset is ~2.8k
  tokens resident in context; independent measurement puts the real cost saving near −10% with an
  interval that touches zero, against −20% advertised, and concentrated on tasks where the agent would
  have over-built. Take the ladder for the code it stops you writing, not for the tokens.

The general rule this illustrates: **an agent brain takes on prose, not dependencies.** A good idea from
a third-party plugin is worth adopting; its update channel and its hooks are not.

## How the reference deployment uses it

In the reference deployment (see [`reference-architecture.md`](reference-architecture.md)):

- the **root-agent (Durin)** runs Dokploy on the VPS, owns the Cloudflare zone (Tunnel + Access in front
  of admin UIs like Dokploy and a Cockpit console), runs the daily backups, and operates the
  `agentic-monitor` heartbeat;
- the **root-agent** owns the application layer since 2026-08-22, but never builds or deploys as
  itself: `npm`, `npx` and `wrangler` run as an unprivileged `builder` Unix user, from a clone of an
  already-pushed commit, with the deploy credentials readable only by that user;
- **Vaultwarden** runs as a Dokploy app as the owner's secret store. Stated honestly, because the
  optimistic version of this line is what an adopter reads while planning recovery: consolidating
  *every* agent secret into it, with an off-box export, is the documented open item — **not yet the
  achieved state**. Today the agents' own tokens still live in per-user files on the box and are
  restored by hand on a rebuild. See [`secrets.md`](secrets.md) and the same note in
  [`reference-architecture.md`](reference-architecture.md).

## Why this is docs-only

Dokploy/Cloudflare/app configs are specific to your domains, providers, and services, and they change
often. Templating them would be brittle and would tempt you to commit environment specifics. The stable,
reusable parts — the *agent + host* layer and the *ownership boundary* above — are what this repo ships.
Wire the app layer to taste; keep its secrets in your store, not in Git.
