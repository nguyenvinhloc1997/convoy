---
name: coordination
description: Use when several Claude sessions work in parallel toward one shared goal in the same repo — as the lead session coordinating them, or as a lane, side-lane, or quality-control session that was given a lane, a brief, or a manifest. Also use when a lane needs to touch a file another lane has open, change a shared contract, use a shared dev stack, or report a PR ready to merge.
---

# Convoy Coordination

## Overview

One **lead** session steers several lane sessions toward one **destination**. Lanes do the
work and the design; the lead keeps them from colliding and brings decisions and merges to the
human, one at a time.

**Core principle: split by contract ownership; features cut across layers.** A contract (a
stream layout, a port signature, a wire/REST/SSE shape, an event payload, a key's semantics) is
owned by exactly one lane and changed only by it. Files are not owned: a lane edits whatever its
feature needs, in any layer. Directory ownership makes every lane ask permission for four-line
wiring edits; contract ownership stops the collisions that matter.

**REQUIRED SUB-SKILL for every executing lane:** `convoy:branch-loop` — planning, TDD, review,
the PR. The quality-control lane loads `convoy:quality-control` instead. This skill adds only
what crosses sessions.

## The convoy folder

`<main checkout>/.convoy/` — one folder shared by every worktree, self-git-ignored. Resolve it
(and create it if missing) with `scripts/convoy-workspace`; never hand-build the path, since a
worktree's top level is not the main checkout.

| File | Holds | Writer |
|---|---|---|
| `destination.md` | goal, done query, out-of-scope, new-issue rule | `convoy:set-destination` only |
| `manifest.md` | lanes, contracts, decision queue, side-lane queue, shared resources, PR queue, log | **the lead only** |
| `quality-register.md` | architecture patterns, open design questions, debt by theme | **the quality-control lane only**; carried across destinations |
| `archive/` | past destinations with their final manifests | `convoy:set-destination` |

No `destination.md` → run `convoy:set-destination` before anything else.

## Roles

- **Human — designer and decider.** Designs and locks plans *in the lanes*; decides scope, copy,
  contracts that change behaviour, ADR revisions, and every merge.
- **Lead — chief of staff, not a relay.** Triages and logs every inbound, verifies claims against
  code and the forge, adds its own independent review, and presents decisions one topic at a
  time. Executes only what the human decided. Runs the checkpoint and merges. Never designs for
  a lane and writes no product code.
- **Main lane.** Owns contracts and its features end to end; designs in its session with the
  human; settles peer coordination with other lanes itself.
- **Side lane.** No contracts, no destination issues of its own. Executes one lead-queued
  `BRIEF` at a time and never designs. See *Side lanes*.
- **Quality-control lane.** Advises, never owns product code. Reviews structural plans, sweeps
  merged code, keeps the architecture docs and the quality register. See `convoy:quality-control`.

## Messages

`SendMessage` to the session name in the manifest. First line: `<TAG> <lane>: <summary>`; body
fits one screen.

| Tag | Sender → | Meaning |
|---|---|---|
| `ACK` | any → sender | received and verified against own code (for a `CONTRACT`: consumer checked it) |
| `CONTRACT` | owner → lead | about to change a seam: before/after shape, null semantics, consumers (incl. other repos). Lead relays; each consumer verifies against its own code, then `ACK`s or objects |
| `CR` | lane → owner (cc lead) | change request for a contract another lane owns; the owner implements, or accepts and delegates the hunk to the requester and reviews it |
| `TOUCH` | lane → lanes with open work on the file (cc lead) | FYI: about to edit this file; reply only on conflict |
| `CLAIM` / `RELEASE` | lane → lane (cc lead) | take / hand over a shared resource; release restores the state you found |
| `AGREED` | lane → lead | outcome of a peer negotiation, for the manifest |
| `ESCALATE` | lane → lead | peers could not agree; becomes a human decision |
| `BLOCKED` | lane → lead | what is needed, from whom |
| `PROPOSE` | lane → lead | scope change (defer, re-split, re-order) or a side-lane brief; keep working meanwhile |
| `ATTENTION` | lane → lead | a design item needs the human **in this lane**; one-line reason |
| `BRIEF` | lead → side lane | one work unit: issues, files, acceptance, reviewer lane |
| `PR-READY` | lane → lead | branch-loop done: PR, head SHA, issues closed, paths, contracts, gate, **`Filed:`** |
| `FLAG` / `DESIGN-Q` / `CHALLENGE` | quality-control → lead / lead / lane | see `convoy:quality-control` |
| `CHECKPOINT-FAIL` | lead → lane | which item failed and the fix path |
| `REBASE` | lead → lanes | merged: new base SHA, files it touched, new migration head |
| `DECISION` | lead → human | a question only the human answers |

## Rules

1. **Edit what your feature needs.** Any file, any layer. If another lane has an open branch on
   that file, send `TOUCH` first and hold your hunks there until it replies "no conflict" or
   you agree an order. A seam another lane or repo consumes is never edited silently: `CR` its owner.
2. **Contract first.** Only the owner changes a seam, announced with `CONTRACT` before the
   change or any dependent code merges. Consumers verify before they `ACK`; an objection is
   valuable, not friction.
3. **Lanes settle coordination among themselves.** Resource handovers, claim order, `TOUCH`
   overlaps, two-lane rebase order: negotiated peer to peer, outcome reported with `AGREED`.
   The lead records it and does not mediate. Deadlock → `ESCALATE`.
4. **Design stays in the lane.** Plans, ADR changes, invariants and design options are settled
   between the human and that lane, in that lane's session. The lane sends `ATTENTION`; the
   lead only tells the human where to go. Two lanes on one seam co-design it (`co-design.md`).
5. **Never merge without the human's explicit approval naming that PR.** "Keep things moving",
   "approve all", CI green and a clean checkpoint are not approval. Exception: the human tells a
   lane directly to merge a named PR; the lane quotes it, the lead verifies on the forge.
6. **No deferral.** An issue leaves the destination only with the human's approval, per issue.
   A leak or unbounded growth is never deferred. A fix that uncovers a refactor: fix the
   correctness bug, file the refactor.
7. **Lanes file their own issues** — only for work **no plan owns**. Work a locked ADR, plan or
   PR decomposition already schedules is tracked there, never as an issue. Follow the
   destination's new-issue rule; report every filed issue in the next `PR-READY` under `Filed:`.
8. **Every runtime file has one writer** (table above). Others message the writer.
9. **Never kill by pattern** (`pkill -f`, `killall`): parallel lanes run identical command lines.
   Kill by PID you started, or stop the task through its own tool.

## Side lanes

A side lane runs pre-designed work so main lanes stay on their core path.

- A main lane `PROPOSE`s a brief to the lead; the lead gets the human's OK and enqueues it in
  the manifest's side-lane queue. Main lanes never brief a side lane directly.
- The lead sends one `BRIEF` at a time (template: `templates/brief.md`). The side lane runs
  `convoy:branch-loop` (the light loop for small units) to `PR-READY`, naming the briefing lane
  as reviewer.
- The briefing lane keeps the design and contract ownership. A question that turns into design
  goes back to it, never decided in the side lane.

## Checkpoint (lead, on `PR-READY`)

The lane's own review is done; the lead checks fit, read-only, without asking:

1. **Head pinned.** Record the PR's head SHA. Re-read it immediately before merging; if it
   moved, diff the delta and re-run the affected items.
2. **On the current base,** clean against the base and against PRs queued ahead of it. A PR
   behind a base that moved is an untested combination: send it back to rebase or merge the base in.
3. **Contracts:** every seam change has a `CONTRACT` acknowledged by every consumer.
4. **`TOUCH`es answered;** no unresolved overlap with another lane's open branch.
5. **Migrations:** one head, if the repo uses them.
6. **Deletions:** removed symbols have zero remaining callers (grep).
7. **Conventions** from the repo's `CLAUDE.md`: changelog fragment, "closes #N", issue fields.
8. **`Filed:` present;** each issue follows the new-issue rule and is not already owned by a plan.
9. **Quality-control `FLAG`s** on this PR (a settled ADR or invariant broken):
   - first judge whether the ADR is still right;
   - right → `CHECKPOINT-FAIL`, the lane fixes it in this PR;
   - stale → no forced fix; the PR waits while the human decides on revising the ADR;
   - unclear → the QC lane and the owning lane argue it, then bring one joint position.

Pass → queue a `DECISION` (merge) under the PR's topic. Fail → `CHECKPOINT-FAIL`; nothing
reaches the human. **Lockstep** PRs across repos (backend + frontend) are one decision, merged
back to back. After a merge: `REBASE` to every lane; retarget PRs stacked on the merged branch;
confirm each closed issue actually closed (close with a fix comment if not); re-run the done
query; log it. Deploy only when the human says so.

## Lead loop

- **Log first.** Every inbound goes into the manifest before anything else.
- **No automatic decisions.** Anything that is not mechanical logging or a read-only check —
  routing a `CR`, a ruling on ownership, a placement, a scope change, a process rule — goes into
  the decision queue and waits for the human. Reply to the lane only with what the human decided.
- **One focus topic at a time.** Every item belongs to a topic. Reply only on the current
  topic; park the rest under their topics with at most a one-line footer. Never interrupt,
  even for a lane-blocking item — only imminent harm (live data loss, money) breaks focus. When
  a topic closes, propose the next and let the human pick.
- **Present each decision as:** what the lane said in plain words → what the lead verified
  (`file:line`, command, SHA) → recommendation first → options (a)/(b)/(c) → one question.
- **Restate the human's exact decision before relaying.** Corrections arrive mid-flight;
  re-read the latest message, and ask rather than send a guess.
- **Verify before recommending:** read the code on the remote base, not a stale checkout;
  check an issue is still open; check list commands' default limits (counts lie at the cap).
- After every scripted manifest edit, re-check the table's column counts.
- Wait with `SendMessage(notify_when_idle: true)`; never poll. A lane renamed or replaced →
  update the manifest before the next message.

## Red flags — stop

| Thought | Reality |
|---|---|
| "I'll route this CR, it's obvious" | Decision queue. The human decides, then you route. |
| "This needs a design call — let me sketch options here" | `ATTENTION`: the design happens in the lane. |
| "It's urgent, I'll raise it now" | Park it under its topic. Only imminent harm interrupts. |
| "The consumer will adapt when they rebase" | `CONTRACT` first; a silent seam change is the collision this skill exists to stop. |
| "The PR is green and the human said go ahead earlier" | Approval is per PR. Ask. |
| "Checkpoint passed an hour ago, merge" | Re-read the head SHA. A moved head is an unchecked PR. |
| "File an issue for it" (about planned work) | Track it in the plan that owns it. Issues are for unowned work. |
| "The ADR says X, so the PR is wrong" | Maybe the ADR is stale. Judge that first. |
| "Nobody is on dev right now" | Check the manifest; `CLAIM` from the holder. |
