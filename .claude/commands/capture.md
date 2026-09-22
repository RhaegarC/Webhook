---
description: Capture a feature spec or bug report into docs/ (requirements & bug capture workflow)
argument-hint: "[feature|bug] [NN]"
---

You are executing the TrailBlaze `/capture` command — the Requirements & Bug Capture workflow.
It creates the spec files that the feature and bug pipelines then consume: a prioritized
idea becomes `docs/features/NN-name.md`; a reported bug becomes `docs/bugs/NN-name.md`. Run it
**before** feature selection (Phase 1 of the `tdd-implement` agent) or bug triage (the `bug-fix`
agent).

`$ARGUMENTS` is a hint like `feature 04` or `bug 02`. Clarify with the user what is being
captured.

1. **Clarify** the item before writing anything — the artifact depends on it:
   - **Feature** → use the `/grill-me` skill to stress-test the idea into scope, acceptance criteria, and success metrics.
   - **Bug** → run a triage checklist, not grilling (repro steps, expected vs actual, severity, environment/version, component), and create/note the Azure DevOps work item ID.
2. **Branch** off `develop`: `docs/feature-[nn]-[name]` or `docs/bug-[nn]-[name]`.
3. **Write the file** using the established template:
   - **Feature** → `docs/features/NN-name.md` following the structure of the existing feature files
     (see [docs/features/archive/01-foundation.md](../../docs/features/archive/01-foundation.md) for the shape: status,
     summary, story, dependencies, acceptance criteria, tests, non-goals). The number claims the priority
     slot — only spec features in implementation order; vague ideas stay in `backlog.md`.
   - **Bug** → follow the bug file template (triage, reproduction, fix plan, close checklist); also add a
     row to `docs/bugs/00-bug-log.md`.
4. **Commit & push**: `docs:` commit type; push the branch.
5. **Create the pull request** (`gh pr create --base develop`) to `develop` quoting the acceptance criteria
   (feature) or the repro + severity (bug). Do NOT merge without approval.
6. **Report** the branch + PR link. After the PR merges, close out (mirrors Phase 8 of the `tdd-implement`
   agent): `git checkout develop && git pull --prune origin develop`, then `git branch -D` the spec branch.

**Every feature file must respect the PRD.** `docs/PRD.md` holds the decisions log, the canonical data
model, and the permission matrix. A feature that contradicts it is either wrong or is proposing a
decision change — raise that explicitly rather than writing a spec that quietly disagrees.
