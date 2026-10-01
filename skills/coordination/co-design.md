# Co-design across a shared seam

Use when two lanes' designs meet on one core path (e.g. a feed producer and the engine commit
path that consumes it). Neither lane designs the seam alone; both sign one decision record.

1. **Split invariants by owner.** Each lane names the invariants it owns on the seam. An
   invariant with no owner is the first gap.
2. **Challenge log.** Each lane attacks the other's proposal with concrete cases. Every entry
   records: the case, the resolution, who conceded. Concessions are the point, not a loss.
3. **Case table — no blanks.** Every issue either lane owns in the area maps to the invariant
   that makes it impossible, or to a named separate fix. A blank row is unfinished design.
4. **Measure the disputed numbers** (latency, cost, capacity) instead of arguing them.
5. **Both lanes sign off → one ADR → the human locks it**, in a lane session. The lead reads it
   independently and returns a gaps list; gaps go back to the lanes for the next revision.
6. **Merge order follows the invariants:** whichever PR establishes an invariant the other relies
   on merges first; the dependent lane rebases onto it.
