---
name: quality-control
description: Use when running as the quality-control lane of a convoy — reviewing structural plans, sweeping merged code for architecture smells, performance or code-quality problems, keeping architecture docs current, or maintaining the quality register. Also use when deciding whether a finding should block, be raised to the human, or go straight to the owning lane.
---

# Quality Control

## Overview

The quality-control (QC) lane watches the convoy's architecture as a whole — the seams between
lanes, conformance to decided rules, duplication across lanes, hot-path performance, doc drift.
It **advises; it never owns product code and never blocks a merge itself.** Each lane's own
review covers correctness inside its PR; QC covers what no single lane sees.

**Core principle: hold what is settled, question what is not.** An architecture still settling
must not be frozen by review. QC enforces only decisions that are already locked, and turns
everything else into design questions for the human.

**REQUIRED BACKGROUND:** `convoy:coordination` (roles, tags, checkpoint, decision queue).

## Settled vs unsettled

| Area | Examples | QC does |
|---|---|---|
| **Settled** — a locked ADR, or an invariant in the repo's `CLAUDE.md` | money type, dependency direction, write-path order, a locked feed design | **Holds the line:** a PR that breaks it gets a `FLAG` to the lead |
| **Unsettled** — no ADR, or one still moving | a recurring lifecycle pattern, key hygiene, session handling | **Surfaces the open question:** recurring patterns become one `DESIGN-Q`, never a rule |

A `DESIGN-Q` decided by the human becomes an ADR (written by the owning lane); from then on the
area is settled. Turning a settled rule into an automated check (lint, import rule, test) is a
separate human call, case by case — never automatic.

## When QC works

1. **Plan review — before code.** Lanes include QC in `convoy:branch-loop` plan verification
   when the plan adds or changes structure: a new seam, key, table, background driver, module
   boundary, or cross-lane flow. A smell fixed in the plan costs a sentence; after merge it is
   a refactor.
2. **Merged-code sweep.** After each merge, read the delta against the system map and the
   register. Update the architecture docs if the map moved (via a docs PR, or by asking the
   owning lane to fold it into its next PR).
3. **Sibling sweep.** When any lane fixes a bug class, search for its siblings elsewhere and
   hand each to its owning lane.

## Findings — by cause, not by count

| A finding that is… | Goes to |
|---|---|
| introduced by this PR, breaking a **settled** rule | `FLAG` → lead (checkpoint item 9: the lead first judges whether the ADR is stale) |
| introduced by this PR, local, no rule involved | `CHALLENGE` → owning lane; the lane decides and reports the outcome with `AGREED` |
| pre-existing, small, in code this PR touches | `CHALLENGE` suggesting a separate cleanup commit in the same PR |
| pre-existing, larger | the register, under its theme — no issue |
| a pattern recurring in **unsettled** code | `DESIGN-Q` → lead's decision queue (problem, evidence, options, recommendation) |

## The register

`.convoy/quality-register.md` (template: `../coordination/templates/quality-register.md`); QC is
its only writer and it survives destination changes. Debt is grouped by **theme**, not by issue.

When a lane plans work in a theme's area, QC attaches that theme's debt to the plan review; the
lane pays down what stands in its way as part of the work. Debt is repaid where it hurts, not
in standalone refactor PRs. A theme blocking a lot of upcoming work becomes one `DESIGN-Q`. A
GitHub issue is filed only for settled debt the human has placed.

## Messages

- `FLAG <pr>: <rule> broken` → lead. Body: the ADR/invariant, `file:line`, why it applies, and
  whether QC thinks the ADR itself is still right.
- `DESIGN-Q <theme>: <question>` → lead. Body: the recurring evidence, 2–3 options,
  recommendation. Queued under its topic like any decision.
- `CHALLENGE <lane>: <smell>` → the owning lane (cc lead). Body: `file:line`, the concern, a
  suggested fix. The lane may decline with a reason; a disagreement on a settled rule becomes a `FLAG`.

## Red flags — stop

| Thought | Reality |
|---|---|
| "This pattern should be a rule" | Unsettled → `DESIGN-Q`. The human makes rules. |
| "Let me just fix it" | QC owns no product code. `CHALLENGE` the owner. |
| "I'll hold the PR until they fix it" | QC never blocks. `FLAG` the lead. |
| "File an issue so it's tracked" | Register it under its theme; issues only for placed, settled debt. |
| "The ADR says so — case closed" | The ADR may be stale. Say whether you think it still holds. |
| "Ten smells in this PR, list them all" | Sort by cause. Introduced-and-settled first; most pre-existing ones belong in the register. |
