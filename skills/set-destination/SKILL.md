---
name: set-destination
description: Use when starting a convoy, when the current destination is reached or replaced (a milestone or phase ships and the team moves on), or when the destination's scope, issues, or lanes need updating. Also use when a convoy session finds no destination.md.
---

# Set Destination

## Overview

The destination is what the convoy drives toward; the lanes are how it gets there. This skill
writes both, and nothing else writes `destination.md`. Run it in the **lead** session.
**REQUIRED BACKGROUND:** `convoy:coordination` (folder layout, roles, message tags).

Every write below happens **after the human approves that draft**. Show it, then write. A
general instruction ("set it up", "go ahead") starts the procedure; it does not approve a draft.
The destination is **reached** only when its done query returns nothing.

## Modes

| Invocation | When | Archive? |
|---|---|---|
| `set` (default) | no destination yet, or replacing one | yes, if one exists |
| `update` | same destination; add/remove issues, re-split lanes, change scope | no |

## Procedure

1. **Replacing?** First, with the old destination still live:
   - tell lanes to finish their current unit and stop starting new ones;
   - list open PRs, `CR`s and `CLAIM`s and resolve each (merge, hand over, or release);
   - list every item the old done query still returns; get a human decision on each —
     carry over, re-scope, or close. Nothing carries over silently.
   Then move `destination.md` + `manifest.md` to `archive/<YYYY-MM-DD>-<slug>/`.
   **`quality-register.md` stays** — architecture debt outlives a milestone.
2. **Shape the destination with the human**, one question at a time:
   - the goal in one sentence;
   - a **checkable** done condition (a query that returns the remaining work), never "when it
     feels done";
   - explicit out-of-scope, each item **excluded from the done query**;
   - external consumers of the convoy's contracts (other repos, services), each in or out of scope;
   - the new-issue rule for issues the lanes file (milestone, assignee, required fields);
   - how many sessions the human will run (session limits bound this, not the issue count).
3. **Pull the work** with the done query. For each item, name the code and contracts it changes
   (issue body, a grep, the code graph) — placement is decided by code, not by the title.
4. **Cut lanes by contract and feature** (`convoy:coordination` core principle):
   - group items by the contracts they change; a contract, its producer and its consumers go
     to one lane where possible; features may cut across layers;
   - work that recurs in a separate repo (e.g. a frontend) gets its own lane;
   - two lanes that would change the same contract run **sequentially in one session**;
   - list each cross-lane seam as a contract with its owner and consumers;
   - order each lane: unblockers of other lanes first, then highest risk;
   - optionally add **side lanes** (no contracts, no issues of their own — fed briefs by the
     lead) and one **quality-control lane** (`convoy:quality-control`).
5. **Show the lane table** (lane → kind → session name → owned contracts → ordered issues →
   cross-lane seams). Iterate until approved.
6. **Write** `destination.md` and `manifest.md`: run `../coordination/scripts/convoy-workspace --init`
   (relative to this skill's base directory) to create `.convoy/` with the templates, then fill them.
7. **Draft one kickoff message per lane** and send after the human confirms. A kickoff contains:
   lane name and kind, owned contracts, ordered issues, known seams, the lead's session name, the
   manifest path, and the skills to load — main and side lanes: `convoy:coordination` +
   `convoy:branch-loop`; the QC lane: `convoy:coordination` + `convoy:quality-control`.

`update` runs steps 3–7 on the delta only, then sends each affected lane its changes.

## Common mistakes

- Lanes cut from issue titles, not the contracts they change → two lanes on one seam on day one.
- Lanes cut by directory → every lane asks permission for small wiring edits.
- A done condition that is not a query → the convoy cannot tell it has arrived.
- Replacing a destination and dropping the old leftovers without a human decision each.
- Archiving the quality register with the manifest — it is meant to carry forward.
- Running more lanes than the human's session budget; a lane waiting for capacity is worse
  than a sequenced lane.
