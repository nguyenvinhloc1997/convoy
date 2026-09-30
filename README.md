# convoy

Personal Claude Code plugin for multi-session development. Builds on `superpowers`.

| Skill | Use |
|---|---|
| `convoy:coordination` | lead + lane rules: ownership, message tags, checkpoint, merge authority |
| `convoy:set-destination` | set / update / replace the goal and cut lanes |
| `convoy:branch-loop` | per-branch execution loop each lane runs |

Per-project state lives in `<main checkout>/.convoy/` (self-git-ignored), created by `skills/coordination/scripts/convoy-workspace [--init]`; templates in `skills/coordination/templates/`.

Install: `claude plugin marketplace add ~/projects/convoy && claude plugin install convoy@convoy`.
After editing a skill: bump `version` in both manifests, then `claude plugin marketplace update convoy`.
