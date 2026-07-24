<!-- title: Getting Started (non-technical walkthrough) -->
# Getting Started — a click-by-click walkthrough

This guide is for someone **new to servers**. It expands the six human steps from the
[README](../README.md) into a concrete, do-this-then-that walkthrough, using **Hetzner Cloud** as the
example VPS and **Tailscale** for easy, safe access. If you already run Linux boxes, skim it — the
[README](../README.md) + [`README_AGENT.md`](../README_AGENT.md) are enough for you.

**The plan:** rent a small box → make it reachable the easy way (Tailscale) → lock it down (expose
nothing) → create the agent's user → install Claude Code → hand the rest to the agent. Budget ~€4–5/mo
for the VPS, plus your Claude subscription.

> Throughout, `sudo` means "run as administrator". When a command starts with `sudo`, you'll be acting
> with full control of the box — go slowly and read before you press Enter.

---

## 1. Rent a VPS (Hetzner Cloud example)

1. Create an account at [Hetzner Cloud](https://www.hetzner.com/cloud) and add a payment method.
2. **New project** → open it → **Add Server**.
3. Choose:
   - **Location:** nearest to you.
   - **Image:** a **Debian-based** OS — **Debian 12** or **Ubuntu 24.04 LTS**.
   - **Type:** the smallest shared-vCPU plan is plenty for the agents alone — e.g. **CX22 / CPX11**
     (~2 vCPU, 4 GB RAM, ~€4/mo). If you'll *also* run apps on it later (Dokploy, n8n, a website),
     pick 8 GB (**CX32 / CPX21**).
   - **SSH key:** if you have one, add it (most secure). If not, Hetzner will email you a **root
     password** — that's fine, we'll stop using it shortly.
   - **Name:** e.g. `my-ops-01`. This is your `<VPS_HOST>`.
4. Create the server. Note its **public IP** (you'll need it once, for the first login).

> Keep the **Hetzner web console** in mind: from the server page, the `>_` **Console** button opens a
> browser terminal directly into the box. That's your **break-glass** access if you ever lock yourself
> out over the network — you can't be truly stranded.

## 2. First login

From your laptop's terminal (macOS/Linux: Terminal; Windows: PowerShell or Windows Terminal):

```bash
ssh root@<the-public-IP>
```

Accept the fingerprint, enter the root password (or your key unlocks it). You're in. Update the box once:

```bash
apt update && apt -y upgrade
```

## 3. Make it reachable the easy way — Tailscale

We want to reach the box from any device **without** exposing it to the internet. [Tailscale](https://tailscale.com)
builds a private network ("tailnet") between your devices; the server **dials out** to join it, so
nothing needs to be open inbound. It also gives you SSH with no keys to manage.

On the **server** (still logged in as root):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --ssh
```

It prints a login URL — open it, sign in (Google/GitHub/email), and the box joins your tailnet.
`--ssh` turns on **Tailscale SSH**, so tailnet devices can SSH in by identity, no key files.

On **your laptop and phone**: install the Tailscale app, sign in with the **same account**. Now, from
any of your devices, you can reach the box by its tailnet name:

```bash
ssh root@my-ops-01        # the short name works once Tailscale MagicDNS is on
```

> This is the access path you'll use day-to-day for the occasional command (`claude-topic list`,
> editing a config). Most of the time, though, you'll just *talk to your agent* from the Claude app —
> see the end of this guide.

## 4. Lock it down — expose nothing (Hetzner Firewall)

Because Tailscale (and later, Cloudflare Tunnel for any web apps) **dials out**, the box needs **no
inbound ports open at all**. So we deny everything inbound.

1. Verify Tailscale SSH works **first** (step 3) — so you don't lock yourself out. (And remember the
   Hetzner **Console** button is always there as break-glass.)
2. In Hetzner Cloud → **Firewalls** → **Create Firewall**.
3. **Inbound rules: delete them all** (leave none). **Outbound: leave as default** (allow all).
4. **Apply to** your server.

That's it: the box now accepts *no* unsolicited inbound traffic. You still reach it over Tailscale, and
your agents still reach the internet (git, Claude, package updates) outbound. This is the "minimal
attack surface" the project is built around.

> Optional extra hardening (advanced): disable password SSH on the box itself
> (`PasswordAuthentication no` in `/etc/ssh/sshd_config.d/`), so even the public `:22` — now firewalled
> off anyway — can't be brute-forced. Not required once the firewall denies inbound.

## 5. Create the agent's user

Your agent runs as its own Linux user (not root's login). Pick a short name — this is `<AGENT>`
(e.g. `root-agent`). Still as root on the box:

```bash
adduser root-agent                       # set a password when prompted; fill the rest with Enter
usermod -aG sudo root-agent              # give it administrator rights
loginctl enable-linger root-agent        # keep its sessions running even when nobody is logged in
```

**Why linger?** Without it, the agent's background sessions would stop the moment you log out, and
wouldn't come back after a reboot. Linger keeps them alive 24/7 — which is the whole point.

Let Tailscale reach this user too, then switch to it:

```bash
su - root-agent                          # become the agent user
```

## 6. Install Claude Code and log in

As the **agent user** (the `su - root-agent` shell from step 5), install Claude Code following the
[official instructions](https://docs.claude.com/en/docs/claude-code), then log in once:

```bash
claude            # the first run walks you through signing in to your Claude account
```

Sign in with the account that has your Claude subscription. Once it works, you can quit (`Ctrl-C`).

## 7. The GitHub side (org + a bot account for the agent)

Your agent stores its brain and config in GitHub. Two things to create in a browser:

1. A **GitHub organization** (free) — this is `<ORG>`. (github.com → your profile → *Your
   organizations* → *New organization* → Free plan.)
2. A **separate GitHub account** for the agent (a "bot"/machine user) with permission to create and
   write repos in that org, plus a **token** wired on the box.

The token wiring has a few specifics (accepting invites, `hosts.yml`, git identity) — follow the
detailed runbook:
[`provision-agent-github-access.md`](../templates/kb-agent-shared/runbooks/provision-agent-github-access.md).
When you're done, `gh auth status` should succeed as the agent user.

## 8. Hand the rest to the agent

Now the payoff. As the agent user, clone this project and start a Claude session pointed at it:

```bash
cd ~
git clone https://github.com/Valiant-Codex/agentic-codex.git
cd agentic-codex
claude
```

In that session, say:

> **"Read README_AGENT.md and bootstrap the system."**

The agent will read the procedure, scaffold your org repos from the templates, provision the host, and
bring its own sessions up — pausing to ask you at the marked checkpoints (creating secrets, going live).
You stay in control; it does the fiddly parts.

---

## You're set — what daily life looks like

- **Talk to your agent from anywhere.** Its sessions appear in the Claude app (web / iOS / Android /
  desktop) via Remote Control. That's how you actually use it — from your phone, from your laptop.
- **Shell access is rare.** When you do need it (run a `claude-topic` command, edit
  `/etc/agentic-monitor.env`), `ssh <AGENT>@my-ops-01` over Tailscale — or just ask the agent to do it,
  since it has `sudo`.
- **You'll get pinged if something breaks.** The dead-man's-switch ([monitoring.md](monitoring.md))
  messages you (e.g. on Telegram) when the box, Docker, or an agent session goes down.
- **Keep your keys safe.** Once you set up the Vaultwarden secret store ([secrets.md](secrets.md)),
  back it up **off the box** — it holds the keys you'd need to rebuild after an incident.

Stuck on a step? Every prerequisite here maps to step 0 of
[`README_AGENT.md`](../README_AGENT.md) — and you can paste an error to your agent and ask it what to do.
