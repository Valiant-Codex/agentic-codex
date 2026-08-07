---
name: patch-management
type: skill
title: Patch Management (OS / Dokploy / container images)
description: Operating procedure for keeping the VPS patched — automatic OS security (unattended-upgrades, optional ESM + Livepatch), deliberate Dokploy + container-image updates (backup + review + deploy), surfaced daily by agentic-divergence-check.
tags:
- skill
- updates
- patch-management
- dokploy
- security
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Patch Management

The privileged **root-agent** owns keeping `<VPS_HOST>` patched. **Principle: security updates =
automatic; version/feature bumps = deliberate** (backup + review, with `agentic-monitor` watching).
This is the operating procedure.

## Cadence & visibility

- **Daily** — the updates section of `agentic-divergence-check` (08:00, after the apt-daily-upgrade window) reports OS + Dokploy status to the `agentic-updates`
  healthchecks.io check → Telegram, only when something is *actionable* (pending security, or a Dokploy
  release). A deferred reboot alone is reported, not alarmed.
- **On request** — the root-agent runs the diagnosis pass below over Dokploy and the stacks it runs,
  and reports. Nothing is scheduled and nothing opens pull requests.
- When either surfaces something, the root-agent proposes the deliberate step to the owner.

## Diagnosis pass (read-only)

Four questions. Nothing here mutates anything.

**1. Platform** — `docker service inspect dokploy --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'`
against the project's latest release (or `settings-getUpdateData` via MCP).

**2. Per service: is the deployed compose what the branch says?** Read the inventory from Dokploy's own
database, **naming the columns** — see Safety, this is the one way to ask that does not hand back every
production secret:

```bash
PG=$(sudo docker ps --format '{{.Names}}' | grep '^dokploy-postgres' | head -1)
sudo docker exec "$PG" psql -U dokploy -d dokploy -A -F'|' -c \
 'select name,"appName","composePath","customGitUrl","customGitBranch","autoDeploy","composeStatus" from compose order by name;'
```

Then diff **that service's own directory** between the deployed commit
(`sudo git -C /etc/dokploy/compose/<appName>/code rev-parse HEAD`) and the branch.
**Never report "N commits behind" as the answer**: the working copy trails the whole repo, most of
which is unrelated to that service. In the reference deployment one stack sat 62 commits behind with a
compose file byte-identical to `main`. Only the per-service diff carries information.

Expect a permanently modified `compose.yaml` in every working copy: Dokploy re-serialises the file it
deploys (dropping quotes, comments and blank lines). That is expected dirt, not drift.

**3. Images: is a newer one published?**

```bash
local=$(sudo docker image inspect <image> --format '{{index .RepoDigests 0}}' | sed 's/.*@//')
remote=$(sudo docker buildx imagetools inspect <image> --format '{{.Manifest.Digest}}')
```

**Self-check, and it is not optional: a *pinned* tag must compare equal.** If it does not, the method
is wrong, not the image — stop and fix the method. That check is what caught the obvious first
attempt: `docker manifest inspect --verbose | .[0].Descriptor.digest` returns the *platform-specific*
manifest digest while the local `RepoDigests` entry is the *index* digest, so every image on the box
looked stale, pinned ones included. `buildx imagetools` returns the index digest and compares
correctly. Services running under Docker **Swarm** reference images by digest and have no local image
under the tag, so the comparison returns nothing for them — that is correct, and their updates are
question 1.

**4. Orphans** — containers whose compose project is no longer a Dokploy service, and volumes no
container references. Both are **reports only**: an `idle` service also leaves its volumes
unreferenced, and those are dormant data, not garbage. Deleting a volume is a human-confirm gate.

## OS updates

- Security applies itself (`unattended-upgrades`). On Ubuntu, **Ubuntu Pro ESM** (extended security
  coverage) and **Livepatch** (kernel patches with no reboot) are OPTIONAL add-ons — enable them if you
  want kernel patching without reboots and extended-support coverage. A reboot to a newer kernel is
  deliberate, done in a low-traffic window on the owner's go.
- Apply all pending on demand (owner-directed): `sudo NEEDRESTART_MODE=l apt-get update && sudo
  NEEDRESTART_MODE=l apt-get -y upgrade`. **List-mode** so host services aren't auto-restarted (avoid a
  surprise NetworkManager/dbus/user-manager restart that could disrupt agent topic sessions); note what
  wants restarting and let the deferred reboot activate it. **Never `autoremove` the running kernel.**
- State: `apt list --upgradable` (plus `pro status` and `sudo canonical-livepatch status` if Ubuntu Pro
  is enabled).

## Dokploy update (human-confirm gate — it's the platform)

Only on the owner's go. Backup first; verify after.

1. **Backup config DB:** `sudo docker exec <container-id> pg_dump -U dokploy -d dokploy | sudo tee
   /var/backups/dokploy/dokploy-db-$(date +%F-%H%M%S).sql >/dev/null` — where `<container-id>` is the
   running Dokploy Postgres container. Confirm it has content (`grep -c "CREATE TABLE"`).
2. Check `settings.getUpdateData` (MCP).
3. Trigger `settings.updateServer` (MCP).
4. Poll until the swarm service `dokploy` runs the new image; verify `settings.getDokployVersion` ==
   latest, `curl localhost:3000` = 200, and app/core containers still healthy (**app containers are not
   touched** by a control-plane update).
5. Rollback = redeploy the prior tag + restore the dump.

## Container image update

1. If the tag is pinned, bump it in `apps/<service>/compose.yaml`, commit and push. For a floating tag
   (`:latest`, `:stable`) there is nothing to edit — **a redeploy is itself the update**, which is
   precisely why deploys are deliberate.
2. **Deploy the service** in Dokploy (compose → redeploy for `<appName>`) — `autoDeploy=false`, so
   nothing deploys on the push alone.
3. Verify the container comes back healthy (`agentic-monitor` covers the critical ones). Rollback =
   restore the previous tag/digest + redeploy.

**The pull-first gotcha — floating tags.** A redeploy alone will **not** fetch a newer image:
`docker compose up -d --build` reuses whatever is cached locally when the tag already exists, even
after the registry has moved on. Observed in the reference deployment on an app that was six minor
versions stale — the redeploy reported "done" in about a second and changed nothing. Fix: run
`docker compose -p <appName> -f <composePath> pull <service...>` **first**, then redeploy. Back up the
app's database first if the jump is more than a point release, and watch the startup logs for the
migration run.

## Safety

- Dokploy updates, kernel reboots, and anything that restarts host services are **human-confirm gates**
  (`policies/approval-policy.md`) — inspect/backup, then wait for the owner.
- **Any `compose-*` MCP call that returns the service object hands back `env` in plaintext** —
  database passwords, admin passwords, app keys. This is not limited to reads: `compose-update`
  returns them in its *response*, so a write you had every right to make still spills the secrets into
  whatever transcript or log is capturing the session.
  **So: read service metadata with the named-column SQL query in the diagnosis pass above, and reach
  for `compose-*` only when you actually need to change something.**
  This warning previously said only "never persist them". That was not enough: in the reference
  deployment an agent that had written this very line then leaked three sets of credentials in one
  session, because the instruction described what to do *afterwards* instead of naming the safe way to
  ask. A warning that does not offer the alternative does not change behaviour.
- Dokploy MCP access ⇒ read access to every production secret — a threat-model fact, and the query
  above is how you avoid exercising it. If a credential does pass through, treat it as exposed and
  tell the owner to rotate it.

## Related

- `runbooks/monitoring.md` — `agentic-monitor` + healthchecks.io dead-man's switch, the observability
  sibling.
- `docs/config-model.md` — the versioned-vs-secret config model.
