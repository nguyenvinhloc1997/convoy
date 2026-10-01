---
name: coordination
description: Use when several Claude sessions work in parallel toward one shared goal in the same repo — as the lead session coordinating them, or as a lane, side-lane, or quality-control session that was given a lane, a brief, or a manifest. Also use when a lane is idle or blocked, needs to touch a file another lane has open, change a shared contract, use a shared dev stack, hand work to a side lane, or report a PR ready to merge.
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

**Second principle: no lane sits idle.** Work lives on one board; an idle or blocked lane pulls
the next eligible item, and a main lane with more planned work than it can run sublets tasks to
idle side lanes while it keeps the design.

**REQUIRED SUB-SKILL for every executing lane:** `convoy:branch-loop` — planning, TDD, review,
the PR. The quality-control lane loads `convoy:quality-control` instead. This skill adds only
what crosses sessions.

## The convoy folder

`<main checkout>/.convoy/` — one folder shared by every worktree, self-git-ignored. Resolve it
(and create it if missing) with `scripts/convoy-workspace`; never hand-build the path, since a
worktree's top level is not the main checkout. **One job per file:**

| File | Holds | Writer |
|---|---|---|
| `destination.md` | goal, scope, done query, out-of-scope, new-issue rule | `convoy:set-destination` only |
| `manifest.md` | **structure:** lanes and the contracts they own, contract consumers, shared resources, standing rules | **the lead only** |
| `board.md` | **all work and its state** — items, decisions, CRs/TOUCHes, PRs (see *The board*) | **the lead only**; lanes change it by message |
| `log.md` | append-only events, one line each | **the lead only** |
| `quality-register.md` | architecture patterns, open design questions, debt by theme | **the quality-control lane only**; carried across destinations |
| `archive/` | past destinations with their final manifest, board and log | `convoy:set-destination` |

Lane plans hold the steps *within* a task; GitHub issues hold defect and debt records. Neither
holds delivery state — that is the board. No `destination.md` → run `convoy:set-destination`
before anything else.

## Roles

- **Human — designer and decider.** Designs and locks plans *in the lanes*; decides scope, copy,
  contracts that change behaviour, ADR revisions, which items are READY, and every merge.
- **Lead — chief of staff, not a relay.** Triages and logs every inbound, keeps the board,
  verifies claims against code and the forge, adds its own independent review, and presents
  decisions one topic at a time. Executes only what the human decided. Runs the checkpoint and
  merges. Never designs for a lane and writes no product code.
- **Main lane.** Owns contracts and its features end to end; designs in its session with the
  human; settles peer coordination with other lanes itself. May sublet tasks from its locked
  plan to idle side lanes (see *Subletting*).
- **Side lane.** No contracts. Runs one task at a time — a sublet `BRIEF` from a main lane, a
  lead `BRIEF`, or a READY board item it pulled — and never designs.
- **Quality-control lane.** Owns architecture-scope design (cross-cutting system design and
  code quality): designs those items with the human in its own session and writes their spec or
  ADR; implementation goes to the owning lane or a side lane. Reviews structural plans, sweeps
  merged code, keeps the architecture docs and the quality register. Owns no product code. See
  `convoy:quality-control`.

## Messages

`SendMessage` to the session name in the manifest. First line: `<TAG> <lane>: <summary>`; body
fits one screen.

| Tag | Sender → | Meaning |
|---|---|---|
| `ACK` | any → sender | received and verified against own code (for a `CONTRACT`: consumer checked it) |
| `CONTRACT` | owner → lead | about to change a seam: before/after shape, null semantics, consumers (incl. other repos). Lead relays; each consumer verifies against its own code, then `ACK`s or objects |
| `CR` | lane → owner (cc lead) | change request for a contract another lane owns; the owner implements, or accepts and delegates the hunk to the requester and reviews it |
| `TOUCH` | lane → lanes with open work on the file (cc lead) | FYI: about to edit this file; reply only on conflict |
| `CLAIM` / `RELEASE` | lane → lead (cc the lane concerned) | take / hand back a board item or a shared resource; a release restores the state you found and says where the work stands |
| `AGREED` | lane → lead | outcome of a peer negotiation, for the board |
| `ESCALATE` | lane → lead | peers could not agree; becomes a human decision |
| `BLOCKED` | lane → lead | what is needed, from whom; send it together with a `CLAIM` of other work, or "nothing eligible" |
| `PROPOSE` | lane → lead | scope change (defer, re-split, re-order), a new board item, or a brief outside a locked plan; keep working meanwhile |
| `ATTENTION` | lane → lead | a design item needs the human **in this lane**; one-line reason |
| `BRIEF` | main lane or lead → side lane | one task (`templates/brief.md`); a re-sent `BRIEF` with the same number replaces the old one |
| `RECALL` | main lane → its side lane (cc lead) | take the task back: stop, commit or push what exists, report where it stands |
| `PR-READY` | lane → lead | branch-loop done: PR, head SHA, issues closed, paths, contracts, gate, `Retired:`, `Size:`, `Tests removed:`, `Filed:` |
| `STATUS?` / `STATUS` | lead → all / lane → lead | fixed-shape status report (`templates/status.md`) |
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
   PR decomposition already schedules is tracked there and on the board, never as an issue.
   Follow the destination's new-issue rule; report every filed issue in the next `PR-READY`
   under `Filed:`. An issue's proposed fix is a suggestion: routing sends the symptom, evidence
   and constraints, and the lane root-causes it.
8. **Every runtime file has one writer** (table above). Others message the writer.
9. **Never kill by pattern** (`pkill -f`, `killall`): parallel lanes run identical command lines.
   Kill by PID you started, or stop the task through its own tool.
10. **Idle means pulling.** A lane with no runnable work pulls from the board (see *The pull
    rule*). "Idle" in a `STATUS` or `BLOCKED` always carries a claim or "nothing eligible".

## The board

`board.md` (template: `templates/board.md`) is the one list of all destination work. Each row is
an item — an issue, a plan task the owner put up for others, a CR, a decision — with one state:

| State | Means |
|---|---|
| `NEEDS-DECISION` | waits on the human; carries its topic. The decision queue is these rows |
| `BLOCKED` | waits on a named item, lane or resource (`BLOCKED on #N`) |
| `READY` | the human approved it as runnable by any eligible lane; carries files and constraints |
| `CLAIMED <lane>` | taken, not started |
| `IN-PROGRESS <lane>` | being worked; for a sublet, `<side lane> for <main lane>` |
| `PR-READY` | PR open, head SHA pinned, checkpoint result. The merge queue is these rows, in order |
| `MERGED` | merged at SHA; the row moves to `log.md` at the next cleanup |

**READY is a human decision.** The lead proposes READY candidates in batches (one `DECISION`
per batch); a lane's `PROPOSE` of a new item lands as `NEEDS-DECISION`. Nothing becomes READY
automatically.

### The pull rule

A lane that is idle or blocked takes the **top eligible READY item**:

1. Read the board; pick the first READY item it is eligible for (side lanes: any item with no
   contract change; main lanes: also items inside their contracts).
2. Send `CLAIM`; start at once. The lead records claims in arrival order — first claim wins;
   a later claimant gets "taken" and pulls the next one.
3. One claimed item per lane at a time. Run it with the light loop of `convoy:branch-loop`
   (full loop if it touches durable state, money or concurrency), including R1–R5.
4. Hit design → stop, `RELEASE` the item back as `NEEDS-DECISION` with the question and where
   the work stands. A pulling lane never designs someone else's item.

### The collision guard

An item is **not READY** while it touches a file another lane has an open branch on, unless a
`TOUCH` to that lane came back "no conflict". An item that changes a contract is READY only for
the owner, or for another lane with the owner's `ACK` on the `CR`. The lead checks both before
proposing a batch, and again when recording a `CLAIM`.

## Subletting

A main lane with more planned work than it can run hands tasks to idle side lanes and stays the
designer.

- **What can be sublet:** a task from the main lane's plan **the human has locked**. No
  per-brief approval from the lead or the human is needed; the main lane `BRIEF`s the side lane
  directly (cc lead, who records `IN-PROGRESS <side> for <main>`). Work outside a locked plan
  still goes through the lead as a `PROPOSE`.
- **The main lane keeps** the plan, the design, its contracts, the review and acceptance. The
  side lane executes and never designs: a question that turns into design goes back to the main
  lane, which answers with a re-sent `BRIEF`. Design changes reach the side lane only that way.
- **Base and windows.** The brief names a frozen base (the base branch, or a tagged seam commit
  of the main lane's branch) and a **hunk window** — the files or functions the side lane may
  edit. Stacks go at most one level deep: a sublet builds on the base or on one tagged seam,
  never on another unmerged sublet. When the seam can merge on its own, merge it first instead
  of stacking.
- **Hand-back,** as the brief says: (a) commits the main lane integrates into its own branch, or
  (b) the side lane's own PR, `PR-READY` to the lead with the main lane as reviewer. Either way
  the main lane accepts the work against the brief's acceptance list.
- **Take-back.** The main lane may `RECALL` a task at any time; the side lane stops, pushes what
  exists and reports where it stands.
- **Limits.** A side lane runs one task at a time; a main lane holds at most two side lanes.
  Two main lanes briefing one side lane: the first `CLAIM` the lead recorded wins; the lead
  steps in only on deadlock.
- The sublet task runs `convoy:branch-loop` with R1–R5 like any other.

## Status (lead → `STATUS?`)

Every lane replies in `templates/status.md`'s shape, at most ten lines, facts only. The lead
verifies PR and issue states on the forge, then gives the human **one rollup**: a row per lane
(Now / Next / Waiting), then **Waiting on the human** (deduped, oldest first), **Merge queue**
(PR-READY rows with pinned heads), **Risks**, and the done-query count.

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
10. **Size and retirement present:** `Retired:`, `Size:` (script output, not hand-typed) and
    `Tests removed:` are in the report. Spot-check one `Tests removed:` mapping. A large net
    growth with `Retired: none` is QC's to challenge, not a checkpoint failure.

Pass → the row becomes `PR-READY` with the pinned head, and a `DECISION` (merge) is queued
under the PR's topic, carrying the `Size:` line. Fail → `CHECKPOINT-FAIL`; nothing reaches the
human. **Lockstep** PRs across repos (backend + frontend) are one decision, merged back to back.
After a merge: `REBASE` to every lane; retarget PRs stacked on the merged branch; confirm each
closed issue actually closed (close with a fix comment if not); re-run the done query; re-check
the collision guard for READY items the merge touched; log it. Deploy only when the human says so.

## Lead loop

- **Log first.** Every inbound gets a line in `log.md` and its board change before anything else.
- **No automatic decisions.** Anything that is not mechanical logging, a claim in arrival order
  or a read-only check — routing a `CR`, a ruling on ownership, a placement, a scope change, a
  READY batch, a process rule — goes on the board as `NEEDS-DECISION` and waits for the human.
  Reply to the lane only with what the human decided.
- **One focus topic at a time.** Every item belongs to a topic. Reply only on the current
  topic; park the rest under their topics with at most a one-line footer. Never interrupt,
  even for a lane-blocking item — only imminent harm (live data loss, money) breaks focus. When
  a topic closes, propose the next and let the human pick.
- **Keep READY stocked.** When fewer READY items exist than idle lanes, propose the next batch.
- **Present each decision as:** what the lane said in plain words → what the lead verified
  (`file:line`, command, SHA) → recommendation first → options (a)/(b)/(c) → one question.
- **Restate the human's exact decision before relaying.** Corrections arrive mid-flight;
  re-read the latest message, and ask rather than send a guess.
- **Verify before recommending:** read the code on the remote base, not a stale checkout;
  check an issue is still open; check list commands' default limits (counts lie at the cap).
- After every scripted board or manifest edit, re-check the table's column counts.
- Wait with `SendMessage(notify_when_idle: true)`; never poll. A lane renamed or replaced →
  update the manifest before the next message.

## Red flags — stop

| Thought | Reality |
|---|---|
| "I'll route this CR, it's obvious" | `NEEDS-DECISION`. The human decides, then you route. |
| "This needs a design call — let me sketch options here" | `ATTENTION`: the design happens in the lane. |
| "It's urgent, I'll raise it now" | Park it under its topic. Only imminent harm interrupts. |
| "The consumer will adapt when they rebase" | `CONTRACT` first; a silent seam change is the collision this skill exists to stop. |
| "The PR is green and the human said go ahead earlier" | Approval is per PR. Ask. |
| "Checkpoint passed an hour ago, merge" | Re-read the head SHA. A moved head is an unchecked PR. |
| "File an issue for it" (about planned work) | Track it in the plan and on the board. Issues are for unowned work. |
| "The ADR says X, so the PR is wrong" | Maybe the ADR is stale. Judge that first. |
| "Nobody is on dev right now" | Check the manifest; `CLAIM` from the holder. |
| "I'm blocked, I'll wait" (lane) | Pull the next READY item, or report "nothing eligible". |
| "This item is obviously runnable, mark it READY" (lead) | READY is a human decision, in batches. |
| "The side lane can decide this detail" (main lane) | Design goes out only as a re-sent `BRIEF`. |
| "I'll stack my sublet on the other side lane's branch" | One level deep, on the base or a tagged seam. |
