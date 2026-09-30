---
name: coordination
description: Use when several Claude sessions work in parallel toward one shared goal in the same repo — as the lead session coordinating them, or as a lane session that was given a lane, owned paths, or a manifest. Also use when a lane needs to touch another lane's files, change a shared contract, use a shared dev stack, or report a PR ready to merge.
---

# Convoy Coordination

## Overview

One **lead** session steers several **lane** sessions toward one **destination**. Lanes do the
work; the lead keeps them from colliding and brings merges to the human.

**Core principle: split by code ownership, never by issue.** A contract (a stream schema, a
port, a wire DTO) is owned by exactly one lane and changed only by it. Splitting one contract
across two owners makes one side wait on the other or build against a shape that is about to
change.

**REQUIRED SUB-SKILL for every lane:** `convoy:branch-loop` — it owns planning, TDD, review,
and the PR. This skill adds only what crosses lanes.

## The convoy folder

`<main checkout>/agent-workspace/convoy/` unless `destination.md` names another path. It is
git-ignored scratch, never committed.

| File | Holds | Writer |
|---|---|---|
| `destination.md` | goal, done condition, issue query, scope, human decision items | `convoy:set-destination` only |
| `manifest.md` | lead's session name; lanes → session names → owned paths → issue order; contracts; open CRs; blockers; shared-resource owners; PR queue | **the lead only** |
| `archive/` | past destinations with their final manifests | `convoy:set-destination` |

No `destination.md` → run `convoy:set-destination` before anything else.

## Roles

- **Human:** the only merge authority. Decides product questions and every scope change.
- **Lead:** writes the manifest, routes messages, sets merge order, runs the checkpoint, files
  new issues. Writes no product code.
- **Lane:** owns its paths; runs `convoy:branch-loop` end to end; reports to the lead by name
  (the lead's session name is in `manifest.md`).

## Messages

Send with `SendMessage` to the session name in the manifest. First line: `<TAG> <lane>: <summary>`.

| Tag | Sender → | Meaning |
|---|---|---|
| `CLAIM` / `RELEASE` | lane → lead | take / return a shared resource (dev stack, shared DB, a live box) |
| `CONTRACT` | owner lane → lead | about to change a seam; body = the contract note. Lead relays to consumers and waits for their ack |
| `CR` | lane → lead | change request for another lane's path; lead routes it, the owner implements |
| `BLOCKED` | lane → lead | what is needed, from whom |
| `NEW-ISSUE` | lane → lead | found work (incl. in paths no lane owns); lead files it per `destination.md`'s rule and places it in a lane |
| `PROPOSE` | lane → lead | a scope question (defer an issue, re-split, re-order); lead turns it into a `DECISION`. Keep working the issue meanwhile |
| `PR-READY` | lane → lead | branch-loop finished; body = PR number, issues closed, paths touched, contracts changed |
| `CHECKPOINT-FAIL` | lead → lane | which checkpoint item failed and the fix path |
| `REBASE` | lead → lanes | a PR merged; rebase onto the base branch |
| `DECISION` | lead → human | a question only the human can answer |

Lanes may message each other directly about a contract; the sender then reports the outcome
to the lead so the manifest records it.

## Rules

1. **Edit only your owned paths.** Anything else — including paths no lane owns — is a `CR`
   (or `NEW-ISSUE`), however small, however urgent, even when the owner looks idle. Deadline
   pressure is a `BLOCKED`, never a direct edit. The owner accepts or refuses a `CR`; the lead records it.
2. **Contract first.** Only the owner changes a seam, and announces it with `CONTRACT` before
   the change itself or any dependent code merges. A non-owner needing a seam change sends a `CR`.
3. **One owner per shared resource**, recorded in the manifest. `CLAIM` before use,
   `RELEASE` after, restored to the state you found it (flatten load, reseed data). A claim on a
   held resource is queued; the lead asks the holder to `RELEASE`. No answer → `DECISION` to the human.
4. **Never merge without the human's explicit approval naming that PR.** Delegation ("keep
   things moving", "you decide", "approve all"), CI green and a clean checkpoint are not
   approval. Exception: the human tells a lane directly to merge a named PR; the lane quotes
   that message in its report, and the lead records it.
5. **No deferral.** An issue leaves the destination only with the human's approval, per
   issue. A fix that uncovers a larger refactor: fix the correctness bug, `NEW-ISSUE` the refactor.
6. **The manifest has one writer.** Lanes never edit it; they message.

## Checkpoint (lead, on `PR-READY`)

The lane's own review is done; the lead does not re-review the code. It checks fit:

1. Every changed path is owned by the lane or covered by an approved `CR`.
2. Every seam change has a `CONTRACT` note acknowledged by its consumers.
3. The branch is on the current base with no conflict against PRs queued ahead of it.
4. It is next in the merge order, or the order is changed on purpose and recorded.
5. It meets the repo's PR conventions (changelog fragments, issue links) from its `CLAUDE.md`.

Pass → a short `DECISION` to the human with the verdict. Fail → `CHECKPOINT-FAIL` to the lane
(e.g. revert the foreign-path change, `CR` it to the owner); nothing reaches the human.
Approved → **the lead merges** per the repo's convention → `REBASE` to the other lanes → the
lane closes its issues, each with a fix comment linking the PR.

## Lead loop

- On each message: update the manifest first, then route.
- Wait with `SendMessage(notify_when_idle: true)`; never poll `ListAgents` or send "are you done?".
- Re-run the destination's issue query at each merge; report progress against the done condition.
- A lane renamed or replaced → update the manifest before the next message.

## Red flags — stop

| Thought | Reality |
|---|---|
| "It's a two-line fix in their file" | `CR`. The owner may be mid-change in that file. |
| "The consumer will adapt when they rebase" | Announce the `CONTRACT` first; a silent seam change is the collision this skill exists to stop. |
| "The PR is green and the human said go ahead earlier" | Approval is per PR. Ask. |
| "This issue is really next phase" | `PROPOSE` it and keep working; only the human moves an issue out. |
| "The owner is idle and it's due today" | `BLOCKED`. Idle is not "not mid-change". |
| "I'll just note it in the manifest myself" (lane) | Message the lead; one writer. |
| "Nobody is on dev right now" | Check the manifest's owner; `CLAIM` first. |
