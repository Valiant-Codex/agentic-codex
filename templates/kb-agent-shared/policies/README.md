---
type: directory-readme
title: Policies Directory README
description: Policies that govern the agent's memory and runtime behavior.
tags:
- policies
- governance
- global
status: active
timestamp: 2026-08-11T00:00:00Z
---
# Policies

## Purpose

Policies that apply to the agent's durable memory, runtime behavior, approvals, and source-of-truth boundaries.

## Files

| File | Type | Status | Purpose | Load when |
|---|---|---|---|---|
| `approval-policy.md` | policy | active | Defines autonomous actions, approval requirements, and forbidden actions. | Before risky/external/irreversible actions. |
| `ssh-access-policy.md` | policy | active | How humans/agents get shell access — Tailscale SSH over the tailnet, no public password auth. | Setting up or changing server SSH access. |
| `github-access-policy.md` | policy | active | Per-agent GitHub bot accounts with least-privilege, matrix-scoped write (org base permission is read; write needs an explicit per-repo grant). | Creating agents or changing repo/token access. |
| `skills-policy.md` | policy | active | How agents author/maintain their own skills — self-writing kept controlled by git, scope, and pruning. | Writing or reviewing agent skills. |
| `memory-policy.md` | policy | active | Defines canonical durable memory and runtime memory rules. Owns the Promotion Workflow. | Updating memory or evaluating memory tools. |
| `source-of-truth-policy.md` | policy | active | Canonical systems for each information class. | Resolving system conflicts. |

The `Type` column above uses the closed vocabulary from
`skills/knowledge-governance-workflow/SKILL.md`. Three rows here used to invent their own values
(`approval-policy`, `memory-policy`, `operating-principles`) — invisible to
`agentic-divergence-check`, which parses frontmatter and not prose tables. A finer distinction goes
in `tags`, never in `type`.

A shipped `operating-principles.md` was **removed** in 0.7.1: 15 principles with no mechanism,
referenced by nothing, and each already binding in a policy, a decision record, a skill, or each
README's own Maintenance Rule — which is the level where such a rule can actually fire.

## Maintenance Rule

Update this README whenever policy files are added, removed, renamed, archived, or superseded.
