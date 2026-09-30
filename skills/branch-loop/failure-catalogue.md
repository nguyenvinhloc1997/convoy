# Failure catalogue

Eight reasoning errors that recur across codebases and languages. Each entry: the error,
how it presents, the detection recipe, and a generic illustration.

**When you find one, sweep the whole path for the class — never patch only the reported
site.** Two of the entries below were found *by* sweeping after fixing a sibling, and one
was introduced *by* the fix for another.

---

## 1. Absence is not evidence

**The error.** A decision rests on the absence of a contrary fact rather than on an
affirmative one. "Nothing needed it", "no error was raised", "the flag wasn't set" are each
produced by two different worlds: *it is genuinely fine*, and *we could not look*.

**How it presents.** A resource is released, a retry is abandoned, a state is declared
clean, because a query came back empty — when the query can also come back empty from an
errored dependency, an empty registry, a cold cache, or a permission failure.

**Detection.** For every branch that concludes something is safe, ask: *what else produces
this same observation?* If one of the answers is "we failed to look", the branch is wrong.

**The fix is structural, not local.** Invert the polarity: make the *safe* action free and
require **positive proof** for the unsafe one. State it so no code path can derive the
unsafe action from a negative. Fixing it at one layer leaves it at the next — it recurs at
the pair level, the discovery level, and the scheduling level independently.

**Asymmetry that settles the design.** Over-retaining costs one idempotent retry.
Over-releasing permanently loses data. When the costs are asymmetric, the default belongs on
the cheap side.

---

## 2. Mirror-shaped and non-discriminating tests

**The error.** A test that reads as verification but cannot fail.

Three shapes:
- **Mirror** — computes its expectation by calling the production code under test.
  `assert f(x) == g(x)` where `f` is `return g(x)`.
- **Non-discriminating** — its fixture sits on the safe side of the distinction, so it passes
  under both correct and incorrect behaviour. It tests something, not the thing it is named
  for.
- **Guard named after a class it does not cover** — a parity/completeness check whose scope
  excludes the case it was built for. Worse than no guard: it stops people looking.

**Detection.** Name the single-line mutation that would make this test fail. If you cannot,
it is not a test. Then **make that mutation and confirm**. A docstring claiming the test
pins an ordering, an invariant, or a contract is a claim to verify, not to believe.

**Illustration.** A test asserted it pinned "the order callers rely on", but its second
assertion sorted both sides, discarding order. Reversing the production ordering broke
nothing anywhere in the suite.

---

## 3. Guard placement follows the consumer, not the caller

**The error.** A guard is placed at the top of a shared function, when the hazard lives in
one specific operation inside it. Callers that never reach that operation are now blocked.

**How it presents.** A validation added to fix a crash starts rejecting legitimate requests.
Typically found as a regression *introduced by a fix*.

**Detection.** For each guard, identify the exact operation that can fail (the division, the
index, the write, the external call) and confirm the guard sits with **it**, not with the
function's entry. Then enumerate every caller and ask which ones reach the guarded operation.

**Illustration.** A zero-divisor check at a function's entry blocked users from closing a
position, because the close path called the same function but never divided.

---

## 4. Per-item isolation in fan-out loops

**The error.** A loop over items (accounts, records, legs, messages, frames) lets one item's
failure abort the whole loop.

**How it presents.** One malformed input stops processing for *everyone*. Especially severe
when the loop performs enforcement, reconciliation, or safety checks — one bad record can
silently disable a control for the entire system.

**Detection.** Every fan-out loop needs per-item isolation. Then, critically:

- **Look inside the handler.** `except` blocks, error paths and audit logs are themselves
  per-item code and can raise.
- **Beware eagerly-evaluated log arguments** over untrusted data. A log line built to record
  that a malformed item was skipped can itself crash on that malformed item.
- **External input is untrusted** — every parse, cast, index, or attribute access on it can
  raise.
- **Know where isolation is deliberately absent, and say so at the site.** A durable-write
  loop may need to stay loud rather than swallow per item. Document it so the next sweep
  does not "fix" it.

**Illustration.** Five instances on one branch. One let a single rejected operation disable
mark-to-market, breach detection and liquidation for an entire market across all accounts.
One was *introduced by the fix* for another, one line away. One was the audit log itself.

---

## 5. Proxies fail open

**The error.** A guard tests a *proxy* for the hazard instead of the hazard itself.

**How it presents.** A marker, flag, or sentinel is used to mean "this state is bad". When
the marker is lost — evicted from a cache, expired, dropped on restart, never written — the
guard reads *safe* and lets the bad state through.

**Detection.** Ask what the guard would read if its input were **missing entirely**, then
ask whether missing is reachable (cache eviction policies, TTLs, partial writes, restarts,
migrations). Prefer testing the **presence of the good state** over the absence of a bad
marker: `EXISTS(state)` fails closed, `NOT EXISTS(bad_marker)` fails open.

**Corollary.** **Allow-lists fail open on new members; deny-lists fail closed.** For
capital, permissions, or safety, enumerate what is *permitted*, not what is forbidden — a
newly-added member is then denied by default rather than silently allowed.

---

## 6. Irreversible bookkeeping before the invalidating check

**The error.** A counter is incremented, a budget consumed, a notification sent, or a record
committed *before* the check that can invalidate the whole operation.

**How it presents.** A retry budget is exhausted by failures that were not the subject's
fault, because the charging step ran before the step that aborts.

**Detection.** Order the sequence: which steps are irreversible, and which can abort the
operation? Every abort must come **before** every irreversible step. When it cannot, the
irreversible step needs to be undoable or idempotent.

---

## 7. Property-level vs symptom-level assertions

**The error.** Tests assert the specific reported symptom, so they catch only their own bug.

**Why it matters.** A test asserting a *property* — "these two things always move together",
"this total never exceeds that", "this set is exactly that set" — catches design errors in
*unrelated future work*. On one branch, a property test written for a crash bug later caught
an entirely different design error in another unit, twice.

**Detection.** When fixing a bug, ask what invariant its existence violated, and assert
**that**, in addition to the reproduction case.

---

## 8. Fail-open composition (cross-unit)

**The error.** Several units each make a locally-defensible "on failure, degrade this way"
choice. Nobody audits what they compose into.

**How it presents.** Every individual decision is correct and defensible. Collectively they
form a systematic bias — e.g. *any* infrastructure defect reduces enforcement — that no
per-unit review can see, because each unit was right.

**Detection.** Periodically enumerate across units:

```
trigger/defect | behaviour on defect | who it favours
```

Then ask two questions:
1. Do these **compose into a bias**?
2. Is any triggering condition **reachable on demand** by someone who benefits from it?

The second question is what turns a composition of safe defaults into an exploit.

---

## Meta-lesson

Three defects on the source branch were introduced by the fixes for other defects, and two
of the worst came from the **instructions given to the implementer** rather than the
implementer's work. When a review finds something, check whether *your own brief* caused it
before attributing it to the person or agent who wrote the code.
