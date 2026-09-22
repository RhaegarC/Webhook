---
description: List all features with their current status
---

List every feature in `docs/features/` (including `docs/features/archive/` if it exists),
excluding the non-features `00-mission-1-sprint.md` and `backlog.md`.

For each feature show:
- **Number + name** (from the filename)
- **Priority** (lower number = higher)
- **Status**: not started / in progress / done & archived
- **Dependencies**

Determine status from:
- Which files are in `docs/features/` vs. `docs/features/archive/`
- Open `feature/*` branches and open pull requests (`gh pr list`) — in progress
- Whether the corresponding branch has been merged to `develop`

Present as a **table sorted by number**, then a one-line "next up" recommendation
(the lowest-numbered feature that is not started and not in progress).

**Flag the deployability gap explicitly.** If 09 is not merged, `develop` is not safe to deploy —
the sequencing note at the top of `docs/features/00-mission-1-sprint.md` states the rule. Say so
plainly rather than letting the table imply otherwise.
