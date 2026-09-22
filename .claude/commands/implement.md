---
description: Implement a feature with TDD (branch → RED → GREEN → verify)
argument-hint: "[number | feature-file | blank]"
---

You are executing the TrailBlaze Scrum+TDD `/implement` command. Follow the TDD workflow encoded
in the `tdd-implement` agent — [.claude/agents/tdd-implement.md](../agents/tdd-implement.md)
(Phases 1–5 below; Phase 0 of the agent is the one-time foundation). `$ARGUMENTS` is optional:
a feature number (e.g. `06`), a feature file path (e.g. `docs/features/06-media-upload.md`),
or blank.

**Resolve the feature target:**
- `$ARGUMENTS` blank → run Phase 1 selection: the next feature is the **lowest-numbered**
  file in `docs/features/` — excluding `00-mission-1-sprint.md`, `backlog.md`, and anything
  under `archive/` — that is not already done, archived, or in progress.
- `$ARGUMENTS` is a number → resolve to the matching file (e.g. `06` → `docs/features/06-media-upload.md`).
- `$ARGUMENTS` is a path → use that feature file directly (must be under `docs/features/`).

**Execute the TDD phases:**

**Phase 1** — Read the feature file completely. Restate its acceptance criteria, success
metrics, and dependencies on lower-numbered features. Verify the feature is not already in
progress (check feature branches and open pull requests) and not archived.

**Phase 2** — Ensure you are on an up-to-date `develop` (`git checkout develop`; `git pull
origin develop`), then create the branch: `git checkout -b feature/<number>-<name>`
(e.g. `feature/06-media-upload`).

**Phase 3 (RED)** — Write failing tests for the acceptance criteria, following the test
tiers in [docs/testing-and-tdd.md](../../docs/testing-and-tdd.md): offline unit tests, a
repository model tier that inspects generated SQL, and the two `Category=Container` tiers
against SQL Edge and Azurite. There is no `IStorageRepository` fake — a storage behaviour
is asserted against a live backend or not at all. Start the containers first
(`docker compose -f docker-compose.test.yml up -d` from `src/api/`); without them those
tiers skip rather than fail, which is not the same as passing.
Run `dotnet test` and confirm the new tests fail for the expected reason.

**Phase 4 (GREEN)** — Implement the minimum code to make the tests pass. Follow existing
code patterns and conventions. Add comments for complex logic.

**Phase 5 (REFACTOR & VERIFY)** — Run the full suite (`dotnet test`) until all tests pass.
Refactor while keeping tests green. If the schema changed, update the data model in
`docs/PRD.md` and the feature file.

**When done:** continue straight through Phases 6–7 — commit, push the branch, and open the
pull request against `develop` — **without stopping to ask for approval first**. Report a
concise summary once the PR exists: what was implemented, test results, changed files, and
the PR URL. Review happens *in the pull request*, not before it — the user reads the diff
there and leaves comments if anything is wrong.

**Merging:** a PR may be merged only once **every** review comment on it has been addressed.
Never merge without explicit approval — a PR that is green and carries no comments is still
not approval to merge.
