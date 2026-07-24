---
type: index
title: Runbooks Index
description: Index of the operational runbooks for running <ORG> agents on a VPS — portability, monitoring, and GitHub access provisioning.
tags:
- runbook
- index
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Runbooks

Operational runbooks for running agents on a VPS. Design rationale lives in the Agentic Codex docs
(`config-model.md`, `monitoring.md`); these are the hands-on procedures.

| Runbook | What it covers |
|---|---|
| [agent-ops-and-portability.md](agent-ops-and-portability.md) | Flagship runbook: versioned-vs-secret split, session resilience via systemd (`claude-topic`), unit gotchas, and fresh-VPS recovery via `provision-agent`. |
| [manage-agents.md](manage-agents.md) | Full agent lifecycle: create (bot + Unix user + runtime + `provision-agent`), manage sessions, and cleanly decommission. |
| [provision-agent-github-access.md](provision-agent-github-access.md) | Per-agent GitHub bot account, least-privilege repo access, and fine-grained token wiring on the host. |
| [fleet-brain-change.md](fleet-brain-change.md) | Apply a coordinated change across several agents' brains without granting any agent standing cross-repo write. |
| [monitoring.md](monitoring.md) | `agentic-monitor` + healthchecks.io dead-man's switch + Telegram alerting — read, test, tune, and reinstall. |
| [patch-management.md](patch-management.md) | Keep the box patched: automatic OS security, deliberate Dokploy/image updates (backup + review). |
