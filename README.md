# convoy

Personal Claude Code plugin for multi-session development. Builds on `superpowers`.

| Skill | Use |
|---|---|
| `convoy:coordination` | lead, main-lane and side-lane rules: contract ownership, message tags, decision queue, checkpoint, merge authority |
| `convoy:quality-control` | the quality-control lane: plan reviews, merged-code sweeps, settled vs unsettled architecture, the quality register |
| `convoy:set-destination` | set / update / replace the goal and cut lanes |
| `convoy:branch-loop` | per-branch execution loop each executing lane runs |

Roles: one **lead**, any number of **main lanes** (own contracts + features), optional **side lanes** (execute lead-queued briefs) and one optional **quality-control lane** (advises; owns no product code).

Per-project state lives in `<main checkout>/.convoy/` (self-git-ignored), created by `skills/coordination/scripts/convoy-workspace [--init]`; templates (destination, manifest, quality register, brief) in `skills/coordination/templates/`. The quality register carries across destinations.

Install: `claude plugin marketplace add nguyenvinhloc1997/convoy && claude plugin install convoy@convoy`.
After editing a skill: bump `version` in both manifests, commit, `git push`, then `claude plugin marketplace update convoy && claude plugin update convoy@convoy` and restart sessions.
