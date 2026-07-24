---
type: runbook
title: Provision Agent GitHub Access (Bot Account + Fine-Grained Token)
description: Step-by-step to give an agent its own GitHub bot account, least-privilege repo access, and a fine-grained token wired on the server.
tags:
- runbook
- github
- tokens
- provisioning
- security
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Provision Agent GitHub Access

Do this per agent (root-agent, cos-agent, dev-agent, future). Steps 1–5 are <OWNER> in the browser;
step 6 is on-box.

## 1. Email — Google Group (free, no seat)

Address pattern `<agent>-agent@<your-domain>`. Create one per agent, e.g. `root-agent@`, `cos-agent@`,
`dev-agent@<your-domain>`.

1. Google Admin / Groups → create group `<agent>-agent@<your-domain>`.
2. Add the owner's mailbox as owner/member.
3. Settings → **Who can post = "Anyone on the web"**, no moderation → so external verification emails (GitHub) are delivered to <OWNER>.
4. Test: email the group from an external address, confirm it lands in <OWNER>'s inbox.

## 2. GitHub bot account

1. Log out of the owner's GitHub account (or use a separate browser / incognito).
2. Sign up at github.com with `<agent>-agent@<your-domain>`. Username e.g. `<bot-username>`.
3. Verify email (link arrives via the group).
4. **Enable 2FA (TOTP)** — store the TOTP seed **and** recovery codes in the password manager. GitHub requires 2FA and this is how <OWNER> keeps control of the bot.
5. Profile name e.g. "Root Agent — <ORG> ops bot".

## 3. Add the bot to its repos (as the org owner)

For each repo in the agent's row of the access matrix: repo → Settings → Collaborators and teams → Add people → invite the bot → role **Write** (RW rows) or **Read** (R rows). The bot must accept the invite (log in as bot → accept).

- **root-agent bot:** `kb-agent-ops-<name>` W, `kb-agent-shared` W, `kb-agent-template` W, `infra` W.
- **cos-agent bot:** `kb-agent-cos-<name>` W, `kb-agent-shared` W, `kb-agent-template` R, `kb-business` W, `kb-intel` W.
- **dev-agent bot:** `kb-agent-dev-<name>` W (once created), `kb-agent-shared` R, `web-main` W, future `web-app` repos W.

(At more agents, prefer a per-agent org **team** with repo access instead of direct collaborators.)

> **Recommended state:** make the bots **org members** (a team `agents`) and set org **base
> permission to `read`**, so every bot can already *read* every repo — the matrix therefore governs
> **write**. Grant write by adding the bot as a collaborator (Write) on exactly its RW repos, or via
> a team. An existing org member does not need to "accept" a repo collaborator add; it takes effect
> immediately.

## 4. Org must allow fine-grained PATs

Org → Settings → Third-party access → Personal access tokens → allow fine-grained tokens, set approval policy. If approval is required, the token request lands in **Pending requests** for an owner (<OWNER>) to approve.

## 5. Create the token (logged in as the bot)

> A **classic PAT** (scopes `repo, workflow`, 90-day expiry) is the simplest option, but fine-grained
> tokens require org membership. The fine-grained recipe below is the tighter alternative, used once
> bots are made org members.

### Fine-grained (tighter alternative)

Settings → Developer settings → Personal access tokens → Fine-grained → Generate:

- **Resource owner:** `<ORG>`.
- **Repository access:** Only select repositories → exactly the agent's repos.
- **Permissions:** Repository → **Contents: Read and write**, **Pull requests: Read and write**, **Metadata: Read-only** (auto). No Administration. Add **Workflows: Read and write** only if the agent edits `.github/workflows/`.
- **Expiration:** 90 days (calendar reminder to rotate). Copy the token.

Repos per bot: root-agent bot → kb-agent-ops-<name>, kb-agent-shared, kb-agent-template, infra · cos-agent bot → kb-agent-cos-<name>, kb-agent-shared, kb-agent-template, kb-business, kb-intel · dev-agent bot → kb-agent-shared, web-main. To add a repo later, edit the token's repo list in place — the token value is unchanged.

## 6. Wire the token on <VPS_HOST> (per the agent's Unix user)

`gh auth login --with-token` **rejects classic tokens without `read:org`**. Wire by writing `hosts.yml` directly. As `<agent>` (via `sudo -u <agent>`):

```bash
mkdir -p ~/.config/gh
cat > ~/.config/gh/hosts.yml <<EOF
github.com:
    users:
        <bot-username>:
            oauth_token: <TOKEN>
            git_protocol: https
    git_protocol: https
    user: <bot-username>
    oauth_token: <TOKEN>
EOF
chmod 600 ~/.config/gh/hosts.yml
cd ~ && git config --global user.name  "<bot-username>"
git config --global user.email "<github-id>+<bot-username>@users.noreply.github.com"
gh api user -q .login                                   # expect <bot-username>
git -C ~/github/<org>/<a-repo> push --dry-run           # smoke test (no side effects)
```

The `gh auth git-credential` helper then serves the token to git; `gh api` uses it too. Use the GitHub `noreply` commit email so commits attribute to the bot without exposing the group address. Run `git config` from the user's `~` (git stats cwd). Never commit the token.

## 7. Revoke the old personal PAT

Once the bot token works on-box: the owner → Settings → Developer settings → Fine-grained tokens → **revoke** any old per-agent PAT (if one was minted under the owner's account). Confirm the agent still pushes/pulls.

## Rotation

On expiry, regenerate the fine-grained PAT as the bot (step 5) and re-run step 6. Nothing else changes.
