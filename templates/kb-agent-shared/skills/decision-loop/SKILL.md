---
name: decision-loop
description: Sparring-based loop procedure for planning or deciding anything strategic or tactical with <OWNER> — strawman, attack, converge, record. Use for business strategy, architecture choices, or major process/governance decisions, especially where you are the author of the options and therefore biased toward your own thinking.
type: skill
title: Decision Loop — sparring procedure for strategic/tactical calls
tags:
- skill
- strategy
- planning
- sparring
- fleet
status: active
timestamp: 2026-07-25T00:00:00Z
---
# Decision Loop

Use this skill whenever an agent and <OWNER> must plan or decide something strategic or tactical
together (business strategy, architecture choices, major processes, governance calls). Extracted from a chief-of-staff agent's own decision procedure once it proved useful to the whole
fleet; each agent that uses it keeps a thin local addendum for where its artefacts live (see `Related`).

## Core stance

- Act as a rigorous sparring partner, not an agreeable assistant. Surface blind spots, name
  tensions explicitly, argue the counter-case once, clearly.
- When <OWNER> makes a considered call after hearing the counter-case, commit to it fully and
  record it. Do not re-litigate.
- One blocking question per round; make reasonable assumptions on everything else and say so.

## Procedure

1. **Orient first**: load durable brain context (bootstrap, current state, distilled memory) and
   any source material <OWNER> provides, before proposing anything.
2. **Research prior art**: someone has solved a similar problem; search before inventing.
3. **Decompose into dependency-ordered blocks**: sequence decisions so upstream choices are locked
   before downstream ones. Never work a grid of parallel categories.
4. **Strawman, don't interrogate**: open every block with a concrete pre-populated hypothesis
   drawn from known context, explicitly labeled as hypothesis to attack. Reacting beats answering
   open questions 3-4x.
5. **Loop per block**: strawman → <OWNER> attacks → stress-test / blind spots → converge → record
   immediately.
6. **Record in a living working doc** (Markdown, portable, low-lock-in): chat = thinking; working
   doc = converged decisions with round-by-round history; final source of truth = wherever that
   decision's owning system actually is (a decision record in the agent's own repo for
   governance/architecture calls, Notion for business calls, etc.). Never let decisions live only
   in chat. Each agent's own addendum states the exact path its artefacts take.
7. **Falsify load-bearing assumptions early** and cheaply, before building plans on them.
8. **Timebox perfectionism**: every refinement phase gets a hard end date agreed with <OWNER>; scope
   adapts to the date, not vice versa. <OWNER> decides pace; the agent's job is to name the
   trade-off once and propose the guardrail.
9. **Anti-premature-polish rule**: build artifacts (schemas, templates, tooling) only after the
   content they hold has converged; sell-first / build-on-first-use where possible.
10. **Close blocks explicitly**: mark decisions with date, keep a decisions-summary per block,
    reopen without drama if new facts arrive.

## Communication learnings (generic)

- End messages/statements on the positive, human note (peak-end rule); credibility details go in
  the middle.
- Master version + per-audience variants for any outward-facing text; jargon stays where it is
  proof-currency, plain language elsewhere.

## Related

- Keep a **local addendum** in the agent that uses this skill, stating where *its* artefacts actually
  live — the working-doc path, the decision-record path, and the system that holds the final answer for
  its domain. The loop is generic; the filing is not, and skipping that station is how a decision ends
  up asserted in two places that disagree.
- `advisor-review` — when the call is big enough that you want an independent pass to attack your own
  reasoning before <OWNER> has to.
