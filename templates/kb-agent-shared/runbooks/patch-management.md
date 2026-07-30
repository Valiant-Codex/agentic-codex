---
type: runbook
title: Patch Management (OS / Dokploy / container images)
description: Operating procedure for keeping the VPS patched — automatic OS security (unattended-upgrades, optional ESM + Livepatch), deliberate Dokploy + container-image updates (backup + review + deploy), surfaced daily by agentic-divergence-check.
tags:
- runbook
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
- **Renovate** opens image-digest-bump PRs on `<ORG>/infra`.
- When either surfaces something, the root-agent proposes the deliberate step to the owner.

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

1. Review the Renovate PR in `infra` (what image, old→new digest/version).
2. Merge it.
3. **Deploy the service** in Dokploy (compose → redeploy for `<appName>`) — `autoDeploy=false`, so
   nothing deploys on the push alone.
4. Verify the container comes back healthy (`agentic-monitor` covers the critical ones). Rollback =
   revert the PR + redeploy.

## Safety

- Dokploy updates, kernel reboots, and anything that restarts host services are **human-confirm gates**
  (`policies/approval-policy.md`) — inspect/backup, then wait for the owner.
- Querying Dokploy (`compose-one`, etc.) returns **service env secrets in plaintext**; never persist
  them (not to git/logs/memory). Dokploy MCP access ⇒ read access to all production secrets — a
  threat-model fact.

## Related

- `runbooks/monitoring.md` — `agentic-monitor` + healthchecks.io dead-man's switch, the observability
  sibling.
- `docs/config-model.md` — the versioned-vs-secret config model.
