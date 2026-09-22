---
description: Select the next feature and implement it (TDD)
---

You are executing the TrailBlaze `/next` command. This is equivalent to `/implement` with no
arguments, using Phase 1 selection from the `tdd-implement` agent — [.claude/agents/tdd-implement.md](../agents/tdd-implement.md).

1. **Select** — determine the next feature: the **lowest-numbered** file in `docs/features/`
   — excluding `00-mission-1-sprint.md`, `backlog.md`, and anything
   under `archive/` — that is not already done, archived, or in progress (check feature
   branches and open pull requests).
2. If there is **no remaining feature**, report that and suggest pruning/adding to
   `docs/features/backlog.md` — then stop.
3. Otherwise, state which feature you are starting (file, number, priority, dependencies),
   then execute the full `/implement` workflow for it (Phases 1–7: branch → RED → GREEN →
   verify → commit → push → pull request). Do not stop for approval before committing —
   review happens in the PR.

**One dependency to watch:** feature 10 (Figma integration) is blocked until the Figma Make
export exists. If selection lands on 10 and the export is not in `src/web/`, say so and stop
rather than inventing a UI.
