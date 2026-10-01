---
name: branch-loop
description: Use when executing a multi-unit plan on one branch over a long or unattended session — many packages/slices, subagent implementers, work spanning hours or days or a context break. Also use for a single risky change to durable state, concurrency, recovery, permissions, or money, where a green suite is not evidence.
---

# Branch Loop

## Overview

Drive a large plan to completion on one branch without losing correctness or coherence, when
the work is too big for one pass and partly delegated to subagents.

**Two principles carry the skill:**

1. **Absence is not evidence.** A decision may rest on an affirmative fact, never on the
   absence of a contrary one. "No test failed", "nothing needed it", "no error appeared" are
   each produced *both* by *it is fine* and by *we could not look*.
2. **The artifact that exists to give you confidence can itself be the broken thing** — the
   test, the guard, the plan, the audit log, the completeness check.

Layers over `superpowers`. It does not replace `writing-plans`, `test-driven-development`, or
`verification-before-completion`. It adds what those lack: what to do when **the command
passes and the test is worthless anyway**, and how to keep a long multi-unit run coherent.

**Invoke the companion `superpowers` skills at each phase — this loop runs *with* them, never
instead of them.** A recurring failure is invoking this loop alone and silently skipping its
complements. Each phase has a required companion:

| Phase | Invoke |
|---|---|
| Design not yet settled | `brainstorming` — before any plan |
| Planning | `writing-plans` — produces the plan this loop executes |
| **Execution (default)** | **`subagent-driven-development` — a fresh implementer per unit + per-task review. This is the default execution vehicle; use it unless the human explicitly asks for inline TDD** |
| Execution (inline, only on explicit request) | `executing-plans` |
| Each unit's implementer | `test-driven-development` |
| Completion | `finishing-a-development-branch` |

This loop supplies the setup gates, the per-unit gate's teeth, the two guardrails, and the
cross-unit coherence pass; the companion supplies the planning, the per-task execution mechanics,
and the finish. **Running one half without the other is a process defect** — if you reach for this
loop and its phase's companion isn't loaded, load it before proceeding.

## When to Use

- A **multi-unit plan on one branch** — packages, slices, phases — especially with subagent
  implementers.
- A run spanning **hours, days, or a context break**, where drift is the real risk.
- **Unattended execution** where the human must trust the output.
- Any **single change** to durable state, concurrency, recovery, permissions, money, or
  deletion — the verification half applies without the orchestration half.

**Skip** for single-file edits, refactors under a suite already mutation-verified, throwaway
scripts, conversational work. Do **not** skip because the suite is green — that is the
condition this exists for.

## The loop

```
ground the plan in the code · verify the plan with fresh eyes    (before the loop)
decompose by dependency
  └─ per unit:
       brief implementer (the plan is fallible)
         → adversarial review on the strong model
         → fix every CONFIRMED finding
         → live verification if observable   (never on a tree under edit)
         → drift reconciliation
         → gate commit recording all five steps, incl. any skip and why
  └─ after the largest unit: re-read governing docs end to end
  └─ at the end: ONE PR to the integration trunk
```

## Setting up

**Align the execution approach with the human before the first unit starts.** A multi-unit run
carries standing rules — the per-unit gate, the regression-testing strategy, how drift against a
ratified decision gets handled, what counts as a trackable issue versus an inline fix, and what
escalates versus what the driver just decides and reports. Confirm all of these as one explicit
handshake up front, rather than letting the human discover the rules mid-run, one surprise at a
time. This comes before, and complements, verifying the plan itself below. In a convoy, the
handshake happens with the human **in this lane's session**, not through the lead.

**Several issues → map root causes before planning fixes.** Group the issues by cause, mark each
cluster as a code smell or a design smell, name the minimal invariant that makes the whole class
impossible, note its ADR impact, and derive the PR order from the clusters. Fixing issues one by
one in title order patches symptoms of the same cause N times.

**Measure before you lock a choice that numbers decide.** Performance designs, venue-parity
behaviour and capacity limits are settled by a bench or a live probe, compared on explicit
criteria, not argued. When two lanes' designs meet on one seam, co-design it
(`../coordination/co-design.md`).

**Ground the plan in the code before you trust it — then verify the plan itself.** Two failures
happen *before* the loop starts, and no per-unit gate catches them:

- **A plan written from the design docs alone re-implements what already exists.** Before the plan
  calls anything "build X", map each element the design asks for to its existing implementation —
  or confirm it is genuinely absent — with a `file:line` for every "reuse" and every "new".
  Re-creating a gate, helper, or piece of state that already exists ships a *second source of truth*
  that then drifts from the first, and that divergence is invisible once execution starts. Where a
  piece already exists, say what each unit *reconciles* (reuse as-is · extend · the doc was wrong),
  not what it rebuilds.
- **The plan is one of the artifacts that can be the broken thing (principle 2).** Verify it against
  the code with fresh eyes before executing — ideally a separate reviewer prompted to break it: does
  every `file:line` it names exist and behave as claimed? Does it miss a unit the design requires, or
  mis-scope a deferral? Would any task, followed literally, red the suite or introduce the divergence
  above? A wrong plan fixed before the first line of code is far cheaper than one found unit-by-unit.

Record the grounding (what exists · what is new · what each unit reconciles) so the implementer
briefs inherit it and the end-of-run coherence pass can check the result against it.

**Both are hard setup gates, not suggestions — and the plan verification is the one most often
skipped.** Do not brief the first implementer until *both* are done and recorded: the grounding map
exists, **and** a *separate* pass has tried to break the plan and its findings are resolved. Writing
the plan and then executing it is the failure — the author reading their own plan is not the review.
The verification is a distinct step with a fresh-eyes reviewer prompted to break the plan (its own
subagent by default); "the plan looks right" is not a substitute. In a convoy, a plan that adds or
changes structure (a seam, key, table, background driver, module boundary) also goes to the
quality-control lane, if one runs, as part of this verification. If the plan changed since it was
verified, it is unverified again. Treat starting unit 1 on an unverified plan as the same class of
error as shipping on a red PRESERVE test.

**Decompose by dependency, not size.** A unit is the smallest thing that can pass a full gate
independently. Correctness fixes later units build on go first; the largest unit goes late
enough that its foundations are verified, but not last — leave room for a coherence pass.

**Brief the implementer that the plan is fallible.** Say so explicitly: the plan may be wrong
and correcting it is expected. Bending code to match a wrong plan is the failure mode. Include
what the plan *doesn't* say — what earlier units fixed, which path actually fires, which
standing rules apply to this unit's shape.

**When a review finds something, suspect your own brief first.** Instructions produce defects
at least as often as implementers do.

## The per-unit gate

All five, recorded in the **commit message** so history is the audit trail rather than your
say-so. A step may be skipped when the rule below says so — never silently.

| # | Step | Rule |
|---|---|---|
| 1 | Full suite, fresh | Run completely, count failures. A pre-change baseline is not evidence |
| 2 | Lint / format / types / architecture | Run each and read its output; these are easy to declare green while red |
| 3 | Schema / migrations | Single head, applied, reversible if the project requires it |
| 4 | Adversarial review | See below |
| 5 | Live verification + drift reconciliation | See below |

### Regression testing strategy

For any change touching existing behavior, build a **behavior ledger before writing code**: each
behavior the unit touches → a bucket → the test(s) that pin it → the one-line mutation that must
still fail it.

- **PRESERVE** — must not change. Its tests are never edited to match new output; a red PRESERVE
  test means the code is wrong, not the test.
- **CHANGE** — deliberately changed. Revise its test only after a new-behavior oracle is written
  and verified red-first against the old code; record the old→new assertion change.
- **NET-NEW** — no prior test. Add oracle tests before or alongside the code.

**Mechanism vs. guarantee:** a test can assert a PRESERVE *guarantee* through a *mechanism* the
unit is changing. Keep the guarantee assertion and swap only the trigger that exercises it — never
delete the guarantee because its mechanism moved. First confirm a guarantee actually rides that
mechanism, though: a test that only pins the mechanism itself, with no guarantee behind it, is a
clean replace.

**On any red test, classify it against the ledger before touching it.** Never set the expected
value to the observed value without that classification — that turns a real regression into a
passing test.

**The PRESERVE set is the run's standing safety net — it stays green through every unit, not just
its own.** On a large or risky rework, the PRESERVE tests are the evidence that the change did not
silently move behavior it was never meant to touch. A PRESERVE test going red is the single most
important signal in the run, and it is a code defect every time, never a test to update.

### 4. Adversarial review — on the strong model

A **separate** reviewer, prompted to **break** the code, not appreciate it:

- Distinguish **CONFIRMED** (executed, or traced exhaustively) from **PLAUSIBLE** (argued
  only). *A finding you cannot substantiate should be dropped, not softened.*
- Rank by real-world impact, not novelty.
- State explicitly where it found **nothing** — silence reads as coverage.
- Give file:line, the triggering input/state, and the resulting wrong output.

Fix every CONFIRMED finding. Re-review non-trivial fixes: a fix is new code.

**Test-quality check (part of the review, not a control run):** for each new test, **name the
one-line mutation that would make it fail. If you cannot, it is not a test.** Three shapes read
as verification but cannot fail — mirror-shaped, non-discriminating, and a guard named after a
class it does not cover; see [failure-catalogue.md](failure-catalogue.md). **Prefer
property-level assertions** — "these two always move together", "this total never exceeds that";
a property written for one bug catches design errors in unrelated later work. (Mutation
*controls* — actually reverting the line or running a mirror test — are **not** part of this
loop; this is a read-only review heuristic.)

### 5. Live verification and drift reconciliation

**Live verification** for anything user-visible or integration-shaped; unit tests do not prove
a wired system works. A unit changing nothing observable at runtime may skip it — state the
skip and its reason in the gate commit.

- **Never verify a tree being edited.** A concurrent agent or file-watcher reload invalidates
  the pass silently.
- **Do not file a defect from one instant's reading.** Divide counts by time; let independently
  updating displays settle before calling them inconsistent.

**Drift reconciliation** — for every decision the unit implements, record exactly one of:
*implemented as decided* · *implemented differently, and the doc now matches* · *the doc was
wrong, and here is the correction*. Silence is not an option.

**Ratified decisions get only two outcomes, not three.** Distinguish a **decision** — a choice
among alternatives the human ratified — from a **description** — a claim about how the code
behaves. Descriptive content uses the three-way reconciliation above. A **ratified decision**
reconciles to exactly *implemented as decided* or **escalate to the human**; *"implemented
differently, and the doc now matches"* and *"the doc was wrong"* are not available for it — either
one would silently let the code override what the human decided. On divergence from a ratified
decision: conform the code, or block the unit and escalate — never edit the decision record to
match the code. This binds the implementer, the driver, and the coherence pass alike, and is the
same shape as the regression rule above: the artifact that exists to catch an error (a PRESERVE
test, a ratified-decision record) must never be rewritten to ratify the error instead (principle 2).

**Compare against the code, not the previous doc** — a doc-to-doc reading reproduces the drift
instead of finding it. Record it in the gate commit and nowhere else; change a design doc only
when the reconciliation says the doc was wrong.

## Standing contracts

**Issues: file only what outlives the branch.** File a defect when it exists in code outside
this branch's diff, or is being deliberately deferred; then fix and close it with evidence in
the same run. The list is a **record, not a backlog**. A finding about your own in-flight work
is a review comment — fix it and record it in the commit message. The durable lesson has a
better home:

| What | Where |
|---|---|
| The **class** of error | [failure-catalogue.md](failure-catalogue.md) |
| The **instance** — file, sites, fix | the commit message |
| An **issue** | only if it outlives the branch, or is deferred |

Never close an issue citing a fix you have not verified exists — including a test you have not
seen.

**Agent isolation.** Concurrent agents get isolated worktrees; file-ownership instructions in
the brief are not sufficient. Sequential agents may share the tree. While any agent is active,
commit with **pathspec** (`git commit -- <paths>`), never `add -A` plus a bare commit.

**Merge authority.** Each unit lands directly on the work branch as it passes its gate. The
branch gets **exactly one PR**, and **the human merges it**. Never squash stacked branches.
**Running as a convoy lane** (a `manifest.md` names you): opening the PR ends your run — send
`PR-READY` to the lead per `convoy:coordination`; the human approves there and the lead merges.

**Autonomy.** Proceed unattended; report at **unit boundaries**. Stop only for a user-facing
rule change, a durable schema or contract change, or something contradicting a ratified
decision. Everything else: decide, record why, keep going — and say what you decided on the
human's behalf when you report.

## Cross-unit checks

**Full coherence pass after the largest unit.** Re-read the governing design docs end to end
against the merged diff, not only the sections that unit touched. Later decisions can
invalidate earlier ratified ones, and review finds this, not implementation.

**Fail-open composition audit.** Once several units have each made a defensible "on failure,
degrade this way" choice, enumerate them together — trigger, behaviour on defect, who it
favours. Then ask whether they compose into a systematic bias, and whether any trigger is
reachable on demand by someone who benefits. Individually correct choices can add up to "any
infrastructure defect disables enforcement", invisible to every per-unit review.

## Red flags — stop

- "The suite is green" offered as evidence for a risky change.
- About to adjust an expectation to match observed output.
- A fix for a *class* of bug that patched only the reported site.
- Concluding something is safe because nothing contradicted it.
- Trusting a doc, plan, or docstring about code you have not read.
- Verifying while something else writes to the tree.
- Filing or closing on a number you have not divided by time.
- Reporting a unit "done" without the five gate steps in the commit.
- Skipping a gate step **silently** — the skip is allowed, hiding it is not.
- Filing an issue for a defect that never left your own branch.
- Briefing the first implementer on a plan no separate pass has tried to break — or one that changed since it was verified.
- Running this loop with its phase companion (`writing-plans`, `subagent-driven-development`, …) not loaded.

**The recurring reasoning errors**, each with a detection recipe:
**[failure-catalogue.md](failure-catalogue.md)**. When you find one, **sweep the whole path for
the class**, never just the reported site.
