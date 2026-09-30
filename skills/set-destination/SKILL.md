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
2. **Shape the destination with the human**, one question at a time:
   - the goal in one sentence;
   - a **checkable** done condition (a query that returns the remaining work, e.g. open issues
     assigned to X on milestone Y), never "when it feels done";
   - explicit out-of-scope;
   - the issue-assignment rule for new issues the lanes find;
   - how many lanes the human will run (session limits bound this, not the issue count).
3. **Pull the work** with the done query. For each item, name the code it will change
   (issue body, a grep, the code graph) — the owner is decided by code, not by the title.
4. **Cut lanes by code ownership** (`convoy:coordination` core principle):
   - group items by the paths and contracts they change;
   - a contract and both its producer and consumer go to one lane;
   - lanes that must share files run **sequentially in one session**, not in parallel;
   - flag every remaining cross-lane path as a pre-declared `CR`;
   - order each lane: unblockers of other lanes first, then highest risk.
5. **Show the lane table** (lane → session name → owned paths → ordered issues → cross-lane
   dependencies) plus the human decision items. Iterate until approved.
6. **Write** `destination.md` and `manifest.md`: run `../coordination/scripts/convoy-workspace --init`
   (relative to this skill's base directory) to create `.convoy/` with the templates, then fill them.
7. **Draft one kickoff message per lane** and send after the human confirms. A kickoff contains:
   lane name, owned paths, ordered issues, pre-declared CRs, the lead's session name, the
   manifest path, and: "Load `convoy:coordination` and `convoy:branch-loop`."

`update` runs steps 3–7 on the delta only, then sends each affected lane its changes.

## Common mistakes

- Lanes cut from issue titles, not the code they change → two lanes in one file on day one.
- A done condition that is not a query → the convoy cannot tell it has arrived.
- Replacing a destination and dropping the old leftovers without a human decision each.
- Running more lanes than the human's session budget; a lane waiting for capacity is worse
  than a sequenced lane.
