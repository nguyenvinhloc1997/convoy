# convoy

Personal Claude Code plugin for multi-session development. Builds on `superpowers`.

| Skill | Use |
|---|---|
| `convoy:coordination` | lead + lane rules: ownership, message tags, checkpoint, merge authority |
| `convoy:set-destination` | set / update / replace the goal and cut lanes |
| `convoy:branch-loop` | per-branch execution loop each lane runs |

Per-project state lives in `<repo>/agent-workspace/convoy/` (git-ignored).

Install: `claude plugin marketplace add ~/projects/convoy && claude plugin install convoy@convoy`.
After editing a skill: bump `version` in both manifests, then `claude plugin marketplace update convoy`.
