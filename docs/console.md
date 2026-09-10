<!-- title: A control console for the fleet (documented, not templated) -->
# A control console for the fleet

Once a fleet outgrows two or three sessions, managing it over SSH stops being pleasant: you cannot
tell from a phone which session is waiting for you, and every lifecycle action means a shell. The
reference deployment answered that with a small web console on the VPS, reachable only over its
Tailscale tailnet. This page is **the pattern and the traps**, not the code.

**The code is deliberately not in this repo.** That is a decision, not an omission — the reasoning is
in [Why this is docs-only](#why-this-is-docs-only) at the bottom. Read that first if you are about to
ask where the app is.

## What the console is for

One sentence: *see which session needs you, and act on it, without a terminal.*

Everything else is optional. In the reference deployment the console also edits the agents' brain
repos, patches the host and shows automation health — but those grew from that one need, and yours
will grow differently. The parts worth copying are the boundaries, not the feature list.

## The shape

```
   your devices on the tailnet
              │  (nothing else can route to it)
              ▼
   tailscaled ── Tailscale Serve, HTTPS, tailnet-only
              │  proxies to 127.0.0.1:<port>, adding identity headers
              ▼
   ┌──────────────────────────────────────────────────────────┐
   │  console process — OWNS the socket                       │
   │   · resolves the client socket's uid                     │
   │   · stamps a per-process token                           │
   │   · then, and only then, hands the request to the app    │
   └──────────────────────────────────────────────────────────┘
              │  every action shells out to an EXISTING verb
              ▼
   claude-topic · systemctl · your host scripts · git — each as the right Unix user
```

Two properties do the work. The console **owns the socket**, so it can learn things about the caller
that no header can forge. And it **calls the verbs you already have** rather than reimplementing
them, so the console can be wrong without the fleet being wrong.

## Identity: what a tailnet does and does not give you

Tailscale Serve adds identity headers to proxied requests — `Tailscale-User-Login` and friends — and
strips any copy the client sent. That is documented and reliable, and it is where most designs stop.

**Stopping there is a mistake on a multi-user box.** Serve proxies to loopback, so from the app's
point of view a genuine request and a forged one are both connections from `127.0.0.1`. Any local
process — including an unprivileged agent that ingests untrusted web content all day — can do:

```
curl -H 'Tailscale-User-Login: you@example.com' http://127.0.0.1:<port>/
```

and it looks exactly like you. On a box where the console can run privileged commands, that is one
HTTP request from a prompt injection to a root action.

The fix is to ask the kernel instead of the client. The process that owns the socket can resolve the
client's uid — read `/proc/net/tcp` for the client's `ip:port` and take the uid column, or use
Tailscale's local API (`tailscale whois`) when the console listens on the tailnet address directly.
`tailscaled` runs as root, so a request that really came through Serve arrives from uid 0; the
injection-exposed agent does not.

Three rules that follow, and each of them was learned rather than assumed:

- **No header, no service.** A missing identity header is a refusal, never "local, therefore
  allowed". The fail-open version of this check is the whole vulnerability.
- **An allowlist of logins, in a root-owned file**, read per request so a change needs no restart —
  and an empty or unreadable file refuses everything.
- **Never trust an internal header from the outside.** See the next section: that is exactly how the
  reference deployment nearly shipped a hole.

## The trap that nearly shipped: a second entry point

The console was built on a Node framework whose Node adapter emits **two** servers: the request
handler the custom wrapper imports, and the adapter's own standalone server. Both run the same
application code, including its identity hook. Only one of them owns the socket.

The standalone one binds `0.0.0.0` by default and forwards the client's headers untouched — so the
uid the app reads is whatever the caller typed. It was sitting in the installed tree, unused, on a
host with a public IP. Started by hand, an unprivileged local user with two forged headers got a
`200`.

Nothing ran it, so nothing was breached. It was still a root console one command away, and the flaw
is instructive because the *application* code was correct throughout: the hole was that the same
code could be reached by a path that had not done the socket work.

Two fixes, and you want both:

1. **The wrapper mints a random token at start**, publishes it where only its own process can see
   it, deletes any client copy of that header, and stamps every request it forwards. The app refuses
   unless the header matches. A server started any other way has no token to offer and fails closed.
   Check the token *before* anything else, so a forged request learns nothing about the allowlist.
2. **Do not ship the other entry point.** The install step deletes the adapter's standalone server
   from the installed tree. One way in.

The general lesson is worth more than the specific bug: **when a check depends on owning the socket,
make it structurally impossible to reach the app without having done it.** Comments and conventions
do not survive a build tool that helpfully gives you a second front door.

## Privilege: the honest version

There is no comfortable answer here, and you should choose deliberately rather than by default.

- **A console that manages agents needs to act as them.** Agent homes are `0750`; their sessions are
  `systemd --user` units. Reading either means being that user or being root.
- **Running the console as root** makes everything work and makes every parsing bug a root bug. The
  reference deployment does this, as an explicitly accepted risk on a single-operator box, recorded
  with its reasoning.
- **Running it unprivileged with a scoped sudoers rule** sounds safer and often is not: a rule broad
  enough to manage agents is broad enough to become root. A narrow rule that constrains nothing is a
  no-op dressed as a boundary.
- **Running it as a dedicated user with no privilege at all** is the genuinely safe option, and it
  buys you a read-only status page. That is a real product; it just is not a control console.

Whichever you pick, write down which one and why. And note the asymmetry: if you run the console as
a non-root user, "the client uid must be root" is the *correct* check and any "…or the server's own
uid" convenience widens it dangerously — the more careful deployment gets the weaker check unless
that path is closed in production.

## Actions: call your verbs, do not reimplement them

The console in the reference deployment performs no fleet logic of its own. Every action shells out:

| It wants to | It runs | As |
|---|---|---|
| list topics with live status | `claude-topic list --json` | each agent |
| topic lifecycle | `claude-topic new / restart / stop / rotate / remove` | that agent |
| force a brain sync | `systemctl start kb-sync.service` | root |
| force a memory mirror | `systemctl start memory-mirror@<agent>.service` | root |
| check for drift | your divergence check | root |
| save a file in a brain repo | `git pull --ff-only` → write → commit → push | the owning agent |

`claude-topic list --json` ([`templates/infra/bin/claude-topic`](../templates/infra/bin/claude-topic))
exists for this: it joins the wrapper's own view of the topics to the live session status the
runtime reports, and publishes both ids a caller needs. It is useful without any console — a status
script can read it too.

Three things this discipline buys:

- **A missing verb is a missing verb, not a special case.** If the console wants something the CLI
  cannot do, add it to the CLI first. Then both have it.
- **The console can be rewritten without touching the fleet.**
- **Every action is reproducible by hand**, which is what you will want at 2 a.m.

Long commands (a package upgrade, a drift check, a redeploy) belong in a small background-job
registry: start it, return an id at once, poll for output. A form action that blocks for four minutes
is a form action that times out.

Two details that cost the reference deployment a round each:

- **Some tools exit non-zero on purpose.** Its divergence checker exits `1` when it *finds* drift —
  its own systemd unit sets `SuccessExitStatus=1` for exactly that reason. Reading "non-zero means
  failed" filled the action log with red for every healthy run. Let a job declare which exit codes
  mean "ran fine".
- **Ids are not interchangeable.** `claude stop|rm` take the runtime's *short* id, not the session
  UUID; given the UUID they answer `No job matching …`. Publish both and use the right one.

## Confirmation, and what to say in it

Anything not recoverable states what is lost, in the button's own words. The rotate action is the
example worth copying: it abandons a conversation and starts a fresh one, and the confirmation says
so, with the session id it is abandoning — because from a phone, one wrong tap is a lost context and
the recovery is hand-editing a state file.

Show the same care in reverse: do **not** demand a confirmation for everything, or they stop being
read.

## Verifying it: serve it and look at it

A type-checker cannot see a page that renders blank, a panel that shows a stale zero, or a layout
whose header row eats the viewport. The reference deployment shipped exactly that bug and only found
it when a human opened the page.

If your console's identity check refuses everything except the operator, a headless browser cannot
reach it either — which is a good check and an awkward harness. The way out is to let the console
accept a client whose uid equals the **server's own** uid, run an instance as the unprivileged build
user, and drive that. In production, where the server runs as root, that test is identical to
"client must be uid 0", so it costs nothing there; make sure it cannot widen anything in a
non-root deployment (see the asymmetry above). Screenshot at a desktop width and a phone width, and
look at both.

## What the reference deployment's console actually has

Listed so you can steal the parts you want and ignore the rest. None of this is required; the first
item is the only one that justified building anything.

- **Home** — every topic of every active agent, grouped by agent, with live status (busy, idle,
  waiting and for what), the context-window fill per session, and the lifecycle actions. A "new
  topic" form. Subscription usage: the rolling 5-hour and 7-day windows with a countdown, taken from
  a snapshot each session's status line writes; no credential is read for it.
- **Files** — one explorer over the brain repos: tree on the left, editor on the right, search
  across every repo. A save is a commit authored by the operator, pushed at once. Saves into the
  shared-skills repo show an extra confirmation naming what they propagate into.
- **Host** — pending system updates and applying them; the container platform's stacks with
  restart/redeploy; forcing the brain sync and the memory mirror, each showing its last run; the
  divergence check with each finding explained in plain English and, where a safe remedy exists, a
  button for it.
- **Workflows** — the automation platform's workflows with their state, last run and recent
  failures.
- **Shell** — a root terminal, one at a time, closing itself after five minutes without traffic.
  The single largest amplifier of any other bug in the product, and worth thinking hard about before
  copying.
- **Log** — every action the console took, with its outcome, searchable.

The host, container-platform and automation panels are specific to that deployment's stack. Yours
will differ, which is precisely why this is a page and not a template.

## Why this is docs-only

The same reasoning as [`app-layer.md`](app-layer.md), plus two of its own:

- **A console is about one operator, not about the framework.** Its repo registry, its agent names,
  its integrations and its whole feature list encode one person's fleet. Roughly 60% of the reference
  implementation's lines name something specific to it. Generalising that is not an edit; it is a
  permanent second artefact to keep in step, and the pieces most worth having — the identity model,
  the verb discipline, the traps above — are exactly the pieces that fit on this page.
- **Shipping a root-privileged web application would change what this repo is.** Everything else here
  is Markdown, shell and systemd units with no dependency tree at all; that is the point of
  "no extra always-on gateway to keep patched" in the README. A console is a thing *you* may choose
  to add on top, with your eyes open. It is not part of the minimal surface this framework describes.

What does cross the line is the framework-level piece: `claude-topic list --json`, in the template,
useful to anything that wants machine-readable fleet state.

## Related

- [`docs/app-layer.md`](app-layer.md) — the same docs-only treatment for the services your agents run
- [`docs/runtime.md`](runtime.md) — the topics/Remote Control model the console drives
- [`docs/config-model.md`](config-model.md) — the portability and security boundary
- [`docs/monitoring.md`](monitoring.md) — the alarm that must keep working whether or not a console exists
