---
type: policy
title: SSH Access Policy
description: How humans and agents authenticate to your servers — Tailscale SSH over the tailnet, no public password auth.
tags:
- policy
- ssh
- tailscale
- access
- security
status: active
timestamp: 2026-07-24T00:00:00Z
---
# SSH Access Policy

## Purpose

Define how the owner and agents obtain shell access to your servers (currently `<VPS_HOST>`), balancing usability across many devices with a safe posture for privileged infrastructure.

## Model

Interactive shell access is via **Tailscale SSH** over the tailnet — authentication is by **device identity** on the owner's tailnet, not by SSH keys or Unix passwords. This is the recommended approach.

- No SSH keypairs to distribute or manage across devices (Mac, Fedora, Windows, Android, iPadOS).
- No Unix passwords exposed to the public internet.
- Access requires the device to be enrolled in the tailnet (`<OWNER>`'s tailnet).

### Tailnet SSH ACL

Governed by the tailnet policy (Tailscale admin console → Access controls → Tailscale SSH). A typical rule allows tailnet members to SSH into their own devices:

- non-root users (`autogroup:nonroot`): `check` mode — periodic browser re-auth (~12h). Change to `accept` only if fully frictionless login is required.
- `root`: `check` mode — always re-auth. Tailscale SSH bypasses OpenSSH, so root is reachable this way regardless of the OpenSSH `PermitRootLogin` setting.

### OpenSSH (host `sshd`) hardening

OpenSSH remains as a fallback but is hardened so it cannot be brute-forced. Config lives in `/etc/ssh/sshd_config.d/10-hardening.conf`:

- `PasswordAuthentication no` — no password login over OpenSSH.
- `KbdInteractiveAuthentication no`.
- `PermitRootLogin prohibit-password` (a.k.a. `without-password`) — root only via key.

Only `root` has an `authorized_keys` entry (break-glass); agent accounts have none, so they are not reachable via public OpenSSH at all — only via Tailscale SSH.

### Optional further hardening

Restrict the public `:22` to the tailnet at the firewall (accept on `tailscale0` + loopback, drop elsewhere). Requires persistence tooling (`iptables-persistent`) on the Docker host. Optional — disabling password auth already removes the brute-force surface; this would only add protection against a hypothetical `sshd` pre-auth vulnerability.

## Per-agent standard

One agent = one Unix user. Agents run locally on the server (not over SSH); SSH is for the owner's interactive access to become an agent's user for inspection/ops. A new agent needs **no SSH setup** beyond its Unix user — the tailnet ACL (`autogroup:nonroot`) already covers it.

## Other channels

- **Cockpit** (`:9090`, reached via Tailscale) is an optional web console that keeps its own PAM password login and is unaffected by this policy. Passwords are stored in the owner's password manager.
- Web services (Traefik `:80`/`:443`, Dokploy, n8n) are independent of SSH and untouched.

## Connecting

Use the MagicDNS FQDN (the bare short name may not resolve on all clients):

    ssh <user>@<VPS_HOST>

Recommended `~/.ssh/config` alias on each device:

    Host vps
        HostName <VPS_HOST>
        User <root-agent>

## Change / approval

Changes to SSH exposure, the tailnet SSH ACL, or `sshd` auth methods are infrastructure changes — see `approval-policy.md` (requires the owner's approval).
