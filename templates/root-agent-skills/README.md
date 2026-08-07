---
type: directory-readme
title: Root-Agent Skills
description: The privileged agent's skills — agent lifecycle, patching, cross-brain changes, monitoring. Copy into whichever agent you make privileged; these do not belong in the shared layer.
tags:
- directory-readme
- skills
- root-agent
status: active
timestamp: 2026-08-07T00:00:00Z
---
# Root-Agent Skills

These are the procedures that need root: creating and decommissioning agents, patching the host,
applying a change across every brain, and running monitoring.

**Copy them into the privileged agent's own `skills/` directory** — not into `kb-agent-shared/skills/`.
Anything in the shared layer is symlinked into *every* agent and its description sits in every agent's
context; a lifecycle procedure that only one agent may execute would be noise for the rest and, worse,
an invitation. Scope is enforced by where a skill lives.

Until 0.7.0 these shipped as `kb-agent-shared/runbooks/*`. That directory is gone: the runtime
discovers skills and discovers nothing else, so a procedure filed anywhere else is reachable only if
some already-loaded text happens to name its path — which, measured across the reference deployment,
one skill in nine ever did.
