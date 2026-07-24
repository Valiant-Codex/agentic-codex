---
type: bootstrap-capsule
title: Shared Bootstrap Capsule
description: Minimal vendor-agnostic bootstrap, ecosystem state, and governance contract shared by all of your org's agents.
tags:
- bootstrap
- shared
- okf
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Shared Bootstrap Capsule

This repository (`kb-agent-shared`) is the **shared governance layer** for all of `<ORG>`'s agents:
global policies, ecosystem state, conventions, cross-agent decisions, and templates. Each agent has
its own repository (`kb-agent-<role>-<name>`) that reaches this repo through a `shared` symlink to a
**sibling clone** at `../kb-agent-shared` (not a git submodule — see `docs/config-model.md`).

Read your **own** agent repo first (its `CLAUDE.md` and `system-prompt.md`), then read from `shared/`
only what the task needs. Keep the initial context small.

## Human

`<OWNER>` is building `<ORG>` and wants pragmatic, portable, low-lock-in systems for AI-assisted
work. They prefer lean knowledge bases, direct language, concrete execution, skeptical analysis, and
one question at a time.

## Ecosystem State (current operating truth)

When older files disagree, prefer this section and the active policies/decisions it links to.

**Runtimes / interfaces**

- **Claude Code** on the VPS (over tmux topic sessions) is the primary/sole operational runtime and
  tool host. See `docs/config-model.md`.
- An interactive reasoning/drafting cockpit can read/write selected GitHub and your business KB when authorized.
- Runtime memories (LLM runtime / vector stores) are caches/working memory, not canonical truth.

**Source of truth**

- Each agent's own `kb-agent-<role>-<name>` repo owns that agent's identity, memory, skills, and tools.
- `kb-agent-shared` (this repo) owns global policies, ecosystem state, conventions, cross-agent decisions, and templates.
- Your business KB (e.g. Notion) is canonical for business knowledge, offers, CRM, and delivery ops — not agent brains.
- Object storage / a shared drive holds raw files/artifacts — not canonical memory.
- `<OWNER>` is the final authority for approvals, strategic direction, and policy changes.

**Agents**

Give each agent a short, memorable name and one of the shared archetypes:

- **cos-agent** — broad, unprivileged Chief of Staff / orchestration agent (`kb-agent-cos-<name>`).
- **root-agent** — narrow, privileged infrastructure/operations agent on `<VPS_HOST>`, runs with sudo (`kb-agent-ops-<name>`).
- **dev-agent** — dedicated build/dev agent for the website, small web apps, and automation
  engineering; self-contained token + deploy rights, never root (`kb-agent-dev-<name>`).
  See `docs/multi-agent-governance.md`.
- Scaffold further agents from `kb-agent-template`.

**Deprecated / historical**

- Retired runtimes and superseded KB-sync layers stay recorded in `decisions/` but do not override this
  section unless explicitly reintroduced.
- Superseded work-management tooling: see `docs/multi-agent-governance.md`. Your current work tracker
  (e.g. Jira/Linear) is the active choice.

## Bootstrap Order

1. Read your agent repo's `CLAUDE.md` and `system-prompt.md`.
2. Read this file (`shared/bootstrap.md`) for ecosystem state and the governance contract.
3. Read `shared/policies/approval-policy.md` before risky actions.
4. Read a specific `shared/decisions/*`, `shared/templates/*`, or one of your own skill/tool/memory files only when the task needs it.
5. Use `shared/index.md` only when navigation is needed.

Do not load whole directories, all decisions, or archived documents at startup.

## Safety

- Never save secrets, credentials, private keys, raw client data, or complete raw logs in any repo.
- Ask before external actions, infrastructure changes, irreversible changes, business commitments, or anything
  that could expose private information. Privileged agents follow their own additional human-confirm gates.

## Discovery Contract

A runtime should locate the shared layer through, in order: an explicit runtime/environment pointer to
`<ORG>/kb-agent-shared`; the agent repo's `shared` symlink to the sibling clone; a repository-root
`bootstrap.md`; or a root `index.md` pointing back to it.
