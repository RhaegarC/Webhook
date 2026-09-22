---
description: Show sprint progress, DoD status, branches, and open pull requests
---

Produce a concise sprint status report for TrailBlaze:

1. **Feature progress** — list every feature in `docs/features/` (01–11) with status
   (not started / in progress / archived), based on file locations and git state. The sprint
   file's feature table is the record you are reporting on, not a second copy of it: if your
   reading and its `Status` column disagree, say so rather than choosing one.
2. **Sprint DoD** — report the Definition of Done items in
   `docs/features/00-mission-1-sprint.md`, checked against what is actually complete. Read the
   criteria from that file; do not carry a list of them here, where it would drift.
3. **Git state** — current branch, recent commits (`git log --oneline -5`), open `feature/*`
   and `fix/*` branches, and any open pull requests (`gh pr list`).
4. **Next actions** — recommend the next feature to pick up and any blocked DoD items.

Present as a short table plus a two-line "next action" summary.

**Two honest caveats to surface whenever they apply:**

- **The deployability gap.** If 09 is not merged, `develop` is not a usable environment — the
  sequencing note at the top of `docs/features/00-mission-1-sprint.md` states the rule and why.
  Say so in the summary rather than reporting a clean-looking sprint.
- **The container tiers, and the difference between a skip and a pass.** The database and storage
  tiers need `docker-compose.test.yml` running; without it they **skip**, and a skip is not a pass.
  Quote the counts from `docs/features/00-mission-1-sprint.md`, which is their only home — a number
  carried here is a number that goes stale here. Report the skipped count alongside the passed
  count, and never describe a run with skips as "all tests passing".
