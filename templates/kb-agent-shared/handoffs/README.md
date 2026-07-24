---
type: directory-readme
title: Cross-Agent Handoffs
description: Lightweight, git-native channel for passing specs and work between your org's agents.
tags:
- handoffs
- multi-agent
- shared
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Cross-Agent Handoffs

A lightweight, git-native channel for passing work between agents — one file per significant handoff,
so deliverables leave a trace instead of living only in a chat window.

Use it when one agent hands another a spec or deliverable that outlives a single message:
cos-agent → dev-agent (a build/automation spec), cos-agent → root-agent (a provisioning request),
or a return handoff reporting an outcome. Trivial back-and-forth does not belong here.

## Naming Convention

```
YYYY-MM-DD_agent-a-agent-b_title-vN.md
```

- `YYYY-MM-DD` — date the handoff is issued.
- `agent-a-agent-b` — from → to, lowercase agent names.
- `title` — short kebab-case subject.
- `vN` — version, bumped when the same handoff is revised.

Example: `2026-07-21_cos-dev_website-design-v1.md`

## File Shape

Keep it short. A handoff should state, at minimum:

- **From / To** — issuing and receiving agent.
- **Status** — `open`, `in-progress`, `done`, or `superseded`.
- **Context** — why this handoff exists.
- **Spec / Ask** — what the receiving agent should build or do, with acceptance criteria.
- **Out of scope / Boundaries** — what stays with another agent.
- **Outcome** — filled in by the receiving agent when done (or a return handoff file).

## Rules

- Handoffs are working coordination, **not** canonical truth. Decisions still land in
  `shared/decisions/`; business knowledge in your business KB (e.g. Notion); each agent's memory in its own repo.
- Never put secrets, credentials, tokens, or raw client data in a handoff.
- Archive or mark `superseded` rather than deleting, so the trail stays intact.
