---
type: policy
title: GitHub Access Policy
description: How agents authenticate to GitHub — dedicated bot accounts with least-privilege, per-repo, fine-grained tokens.
tags:
- policy
- github
- tokens
- access
- security
status: active
timestamp: 2026-07-24T00:00:00Z
---
# GitHub Access Policy

## Purpose

Define how each agent authenticates to your `<ORG>` GitHub org, with least-privilege access scoped to only the repositories it needs.

## Model

Each agent has its **own dedicated GitHub bot account** (machine user), not a token minted from the owner's personal account.

Rationale:

- **Attribution**: the org audit log records the *account* as the actor, not the token name. Distinct bot accounts make each agent's actions traceable; personal PATs all appear as `<owner-github>`.
- **Access ceiling**: a bot is added as collaborator only to its repos, so even a mis-scoped token cannot reach other repos (defense in depth).
- **Future identity**: agents will need their own presence on other platforms (your work tracker, etc.); a per-agent identity + email is the clean foundation.
- **Cost**: on the GitHub **Free** plan, collaborators are free.

GitHub explicitly permits machine/bot accounts in addition to one personal account.

### Bot identity

- **Email**: a Google Group `<agent>-agent@<your-domain>` (Workspace groups are free, do not consume a seat), configured to **accept external mail**; delivers to the owner. Receive-only alternative: a Workspace user alias. Avoid plus-addressing (`owner+agent@`) — some platforms reject `+`.
- One bot GitHub account per agent, added as **collaborator on only its allowlisted repos** with the minimum role (Write / Read).
- Token stored on the server for that agent's Unix user (mode-600 `~/.config/gh/hosts.yml`), never committed.

### Token type

Bots are **organization members** (`<org>-<root-agent>-bot`, `<org>-<cos-agent>-bot`, `<org>-<dev-agent>-bot`), each with per-repo access grants matching the matrix. Two workable token types:

- **Classic PATs** with scope **`repo, workflow`**, 90-day expiry — simplest to wire.
- **Fine-grained PATs** (available once the org is the resource owner) — tighter *per repo*: Resource owner `<ORG>`, Only-select-repositories, permissions **Contents RW + Pull requests RW + Metadata R** (+ Workflows RW only if editing CI). Editing a fine-grained token's repo list later keeps the token value unchanged (no re-wiring).

Either way, a token cannot exceed the bot account's own repo access.

> ### ⚠️ A fine-grained PAT inherits nothing — read before choosing one
>
> Learned by burning a provisioning attempt on it. There are **two independent layers**, and moving the
> account does not move the token:
>
> - an org **base permission of `read` does not reach a fine-grained token**;
> - a **collaborator grant on the bot account does not reach its fine-grained token** either.
>
> Only the token's own *Only-select-repositories* list grants it anything. And the failure mode is
> actively misleading: a fine-grained token with an **empty** selection still authenticates
> (`gh api user` cheerfully returns the bot) and still lists any **public** repo, which reads as
> "the token works". It does not.
>
> **The only test that means anything is a git operation against the target repo:**
>
> ```bash
> GH_TOKEN=<token> gh api repos/<ORG>/<BRAIN> -q .permissions   # 404 = no access at all
> git ls-remote https://<bot>:<token>@github.com/<ORG>/<BRAIN>.git
> #   → "remote: Write access to repository not granted." means exactly what it says
> ```
>
> **The trap that matters for security:** one fine-grained token applies **a single permission set to
> every repository it selects**. You cannot give it Contents RW on the agent's brain and Contents
> **R** on `kb-agent-shared` — so putting both in one token grants the agent **write on the shared
> governance layer**, which every agent auto-loads through `kb-sync`. That is the one grant this whole
> model exists to withhold from injection-exposed agents (see the access matrix note below).
>
> **Therefore: for an agent that needs RW on its own brain and R on `kb-agent-shared` — i.e. the normal
> case — a classic PAT is the correct choice, not a compromise.** A classic token rides the bot
> account's own grants, so it inherits write on its brain and read-only on shared for free, exactly
> matching the matrix. Reach for fine-grained only when a token genuinely needs one repo, or when
> per-repo permission sets exist.
>
> Scope note: `workflow` is only needed to edit `.github/workflows/`. An agent with no CI should get
> **`repo` alone**.

> **Org base permission is `read` — by deliberate choice** (Org → Settings → Member privileges → Base permissions). Every org member — the agent bots and any future member — can **read all repos**. This is accepted for operational simplicity; the risk is low because all repos are internal, **write stays matrix-scoped** (push always needs an explicit per-repo grant), and **secrets never live in git** anyway (see `approval-policy.md`).
>
> Consequence: **the matrix below governs *write*, not read.** Never rely on repo boundaries to hide content from another agent — assume every agent can read every internal repo. If something must be hidden from an agent, it does not belong in an org repo.
>
> Tighter alternative: set base permission to `No permission` and grant read explicitly per repo, at the cost of more per-repo administration.

### Wiring on the server

`gh auth login --with-token` **rejects classic tokens lacking `read:org`**. Wire instead by writing the token directly into that Unix user's `~/.config/gh/hosts.yml` (mode 600) — the standard `gh auth git-credential` helper then serves it to git, and `gh api` uses it too. For example:

```yaml
# ~/.config/gh/hosts.yml  (mode 600, owned by the agent's Unix user)
github.com:
    oauth_token: <TOKEN>
    user: <bot-username>
    git_protocol: https
```

Also set the bot's git identity so commits attribute correctly:

```
git config --global user.name  "<bot-username>"
git config --global user.email "<id>+<bot-username>@users.noreply.github.com"
```

Pending collaborator invites can be accepted non-interactively with the bot's own token: `gh api --method PATCH user/repository_invitations/<id>`.

## Access matrix

Three example archetype agents: **root-agent** (privileged infra/ops, has sudo), **cos-agent** (chief-of-staff, broad, unprivileged), **dev-agent** (build/deploy, own deploy token, never root).

| Repo | root-agent (ops) | cos-agent (cos) | dev-agent (web-dev) | Notes |
|---|---|---|---|---|
| `kb-agent-ops-<root-agent>` | RW | — | — | root-agent's brain |
| `kb-agent-cos-<cos-agent>` | — | RW | — | cos-agent's brain |
| `kb-agent-dev-<dev-agent>` | — | — | RW | dev-agent's brain |
| `kb-agent-shared` | **RW** | R | R | governance **and `shared/skills/`**. The **root agent is the sole direct writer**: this layer carries instructions every agent auto-loads through the unreviewed auto-sync channel, so write here is effectively write into the privileged agent's context, and belongs to the **least**-injection-exposed principal. The cos-agent — which ingests web, email and tickets — reads only, and requests changes. |
| `kb-agent-template` | RW | — | — | the seed brain; the root-agent provisions new agents from it (`manage-agents`). Same argument as the row above: a template is instructions a future agent will load. |
| `infra` | RW | — | — | infra / ops (host layer, root-agent only) |
| `kb-business` | — | RW | — | business KB |
| `web-main` | — | — | RW | website; the dev-agent owns the app layer |
| _future `web-app` repos_ | — | — | RW | small web apps built by the dev-agent |

Read is **org-wide** (base permission `read`); the RW/R/— entries above therefore encode **write** intent — `—` still means "no write", not "no read".

**Applying a change to another agent's brain never requires granting cross-repo write.** The root-agent does it on-box as that agent's Unix user, committing with that agent's **own** bot token; fleet-common content belongs in `kb-agent-shared` (written once, read by all). Broad write on anything an agent auto-loads — the shared layer, the template — is therefore kept **off** the injection-exposed agents by design — see the Agentic Codex docs (`multi-agent-governance.md`).

> The dev-agent also holds a **hosting-platform deploy token** (e.g. Dokploy) scoped to the website project only, and may build automation workflows via an n8n MCP/API. Those are outside this GitHub-only matrix. The dev-agent never has root, never touches Dokploy/Cloudflare/backups (the root-agent's host layer).

### Future agents

- New agent → new bot account + Group email, added as collaborator to its own `kb-agent-<role>-<name>` (RW) + `kb-agent-shared` (RW for KB-maintaining agents, R for build/consumer agents) + `kb-agent-template` (R if it needs to re-scaffold), plus any domain repos it needs. Record additions in this matrix.
- A personal-assistant agent scoped to the owner's personal account (e.g. `<owner-github>/kb-personal` as its brain) lives outside the business org and is not part of this org matrix.

## Migration

If tokens currently live under the owner's personal account, once each `*-bot` account is set up and verified, **revoke the corresponding personal PAT** and switch the server to the bot's token.

## Change / approval

Adding accounts, changing repo access, or changing token scope are access changes — see `approval-policy.md`.
