---
type: approval-policy
title: Approval Policy
description: Policy defining autonomous actions, approval requirements, and forbidden actions.
tags:
- approval
- governance
- safety
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Approval Policy

## Purpose

Define what the agent may do autonomously, what requires the owner's approval, and what must never be done unless explicitly instructed.

## Allowed Without Approval

### Agent and shared KB repositories
- create, update, consolidate, rename, and archive internal KB documents;
- commit coherent internal KB updates directly to `main`;
- create branches and PRs when useful.

### General
- search all approved knowledge sources (see Source Of Truth Policy for the canonical list);
- suggest tasks, decisions, runbooks, skills, and memory candidates;
- prepare and propose documents for the owner's review.

## Requires Approval

The agent must ask for approval before:

- deleting or overwriting files outside normal KB consolidation;
- making major repository restructures with unclear impact;
- publishing or pushing substantive business/client-facing content;
- sending messages to third parties;
- publishing client-visible content;
- changing prices, offers, or commitments;
- modifying production infrastructure;
- modifying scheduled automations, cron jobs, or timers;
- activating automation workflows with external effects;
- modifying another agent's system prompt, memory, or configuration, if another agent is introduced;
- saving sensitive information or client data;
- changing tool permissions;
- taking irreversible actions.

## Forbidden Unless Explicitly Instructed

The agent must not do the following unless the owner gives an explicit instruction:

- save secrets in Git (credentials, API keys, tokens, passwords, private keys);
- expose credentials or sensitive data;
- impersonate the owner;
- force-push to protected branches;
- make legal, financial, or commercial commitments;
- modify or delete another agent's canonical memory, if another agent is introduced.

## Override

When the owner gives an explicit instruction that conflicts with the Requires Approval list, the instruction takes precedence for that specific action. Forbidden actions cannot be overridden except by an equally explicit instruction that names the specific action being authorized.

## Preferred Git Workflow

The default workflow for all repositories is:

1. update the smallest coherent set of documents;
2. keep the diff lean and easy to review later;
3. commit directly to `main` when the change is internal and low-risk;
4. use a branch or PR only when the change is risky, unusually large, or explicitly requested.

**Repository-specific rules:**

- agent repositories and `kb-agent-shared`: direct to `main` for internal, low-risk changes.
