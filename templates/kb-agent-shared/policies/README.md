---
type: directory-readme
title: Policies Directory README
description: Policies that govern the agent's memory and runtime behavior.
tags:
- policies
- governance
- global
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Policies

## Purpose

Policies that apply to the agent's durable memory, runtime behavior, approvals, and source-of-truth boundaries.

## Files

| File | Type | Status | Purpose | Load when |
|---|---|---|---|---|
| `approval-policy.md` | approval-policy | active | Defines autonomous actions, approval requirements, and forbidden actions. | Before risky/external/irreversible actions. |
| `ssh-access-policy.md` | policy | active | How humans/agents get shell access — Tailscale SSH over the tailnet, no public password auth. | Setting up or changing server SSH access. |
| `github-access-policy.md` | policy | active | Per-agent GitHub bot accounts with least-privilege, matrix-scoped write (org base permission is read; write needs an explicit per-repo grant). | Creating agents or changing repo/token access. |
| `skills-policy.md` | policy | active | How agents author/maintain their own skills — self-writing kept controlled by git, scope, and pruning. | Writing or reviewing agent skills. |
| `memory-policy.md` | memory-policy | active | Defines canonical durable memory and runtime memory rules. | Updating memory or evaluating memory tools. |
| `operating-principles.md` | operating-principles | active | Durable operating principles for all agent work. | Reviewing general behavior. |
| `source-of-truth-policy.md` | policy | active | Canonical systems for each information class. | Resolving system conflicts. |

## Maintenance Rule

Update this README whenever policy files are added, removed, renamed, archived, or superseded.
