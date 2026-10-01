BRIEF <side lane> #<n>: <one-line goal>
From: <briefing lane> (reviewer, keeps the design) · Source: <locked plan + task id> | lead, approved by <human> <date>

Issues: #<n> — closes / part of
Base: <base branch @ sha> | <seam tag on the briefing lane's branch>
Window: <files / functions the side lane may edit>; anything else → ask the briefing lane
Out of scope: <what not to change>
Acceptance:
- <checkable outcome, e.g. a command that must exit 0, an oracle test that must go red first>
Contracts: none | <contract> (owner: <lane>)
Loop: light | full `convoy:branch-loop`, with R1–R5
Hand-back: (a) commits on <branch> for the briefing lane to integrate | (b) own PR, `PR-READY` to the lead, briefing lane reviews
Report: design questions go to the briefing lane, never decided here; a re-sent BRIEF #<n> replaces this one.
