---
type: directory-readme
title: Decisions Directory README
description: Cross-agent / ecosystem decision records for your org's agents.
tags:
- decisions
- governance
- global
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Decisions

## Purpose

Cross-agent / ecosystem decision records: choices that affect more than one agent — the durable memory
model, repository structure, operating model, runtime, security boundaries, or collaboration style.
Decisions that affect only one agent live in that agent's own repository, not here.

Each record is a short, dated document capturing the context, what was decided, the rationale,
alternatives considered, and consequences. Use `templates/decision-template.md` as the starting shape.

This template ships with an **empty** decisions directory — populate it with your own records as your
fleet's architecture evolves.

## Naming Convention

`YYYY-MM-DD-short-description.md`

- `YYYY-MM-DD` — the date the decision was made.
- `short-description` — a kebab-case slug summarizing the decision.

## Loading Rule

Load only the current/relevant decision when a task needs it. Do not load the whole directory at
startup. Superseded and historical records explain why the architecture changed; they should not
override the current ecosystem state in `bootstrap.md` or active policies.

## Maintenance Rule

Update this README (add a `Files` table if the directory grows) whenever a decision is added, removed,
renamed, archived, or superseded. Mark stale decisions `superseded` rather than deleting them, so the
reasoning trail stays intact.
