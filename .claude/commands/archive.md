---
description: Archive a completed feature after its PR is merged to develop
argument-hint: "[number]"
---

You are executing the TrailBlaze `/archive` command (Phase 8 of the `tdd-implement` agent — [.claude/agents/tdd-implement.md](../agents/tdd-implement.md)).
`$ARGUMENTS` is the feature number (e.g. `06`) — resolve it to
`docs/features/<number>-<name>.md`.

**Only archive after the feature PR is merged to `develop`:**
1. **Verify the merge** — confirm the feature's changes are in `origin/develop` (check
   `git branch -a --contains <feature-branch>` on `develop`/`origin/develop`, or that the
   PR is closed/merged). If **not merged, stop** and explain why.
2. Create `docs/features/archive/` if it doesn't exist.
3. Move the file: `git mv docs/features/<file> docs/features/archive/<file>`.
4. Confirm the Phase 1 selection scan now skips the archived feature.
5. Report the archived feature and suggest updating sprint tracking
   (`docs/features/00-mission-1-sprint.md`).

**Before archiving feature 09, check the deployability gap closes.** 09 is the one that makes
`develop` safe (the sequencing note in `docs/features/00-mission-1-sprint.md` states why). If 09 is
being archived, confirm the permission matrix in the PRD is fully covered by merged tests — that is
the evidence, not the feature file's checkboxes.
