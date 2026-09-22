---
description: Add missing TDD tests (RED) for a feature without implementing them
argument-hint: "[number]"
---

You are executing the TrailBlaze `/add-test` command (TDD **RED** step). `$ARGUMENTS` is the
feature number (e.g. `06`) — resolve to `docs/features/<number>-<name>.md` and read it.

1. **Read** the feature's acceptance criteria and its **Tests (TDD)** block.
2. **Inspect** the existing backend test projects under `src/api/` (`TrailBlaze.Api.Test`,
   `TrailBlaze.Repository.Test`, `TrailBlaze.Service.Test`) to see which layer(s) the feature
   touches already cover.
3. **Identify gaps** — acceptance-criteria behaviors not yet covered by a test.
4. **Write NEW failing test(s)** for those gaps, following the test tiers in
   [docs/testing-and-tdd.md](../../docs/testing-and-tdd.md). In outline: offline unit tests for
   validation, policy construction and permission evaluation; a repository model tier that stays
   offline and inspects generated SQL via `ToQueryString()`; and two **`Category=Container`** tiers
   in `TrailBlaze.Repository.Test` that run the real `AzureBlobStorageRepository` against Azurite
   and a real `TrailBlazeContext` against SQL Edge. There is **no `IStorageRepository` fake** —
   nothing stands in for the storage implementation, so a storage behaviour is asserted against a
   live backend or not at all. Those tiers skip, never fail, when nothing answers.
   Place them in the matching layer's test project under `src/api/` (e.g. `TrailBlaze.Service.Test`,
   `TrailBlaze.Api.Test`).
5. **Run the relevant suite** (`dotnet test`) and confirm
   the new tests **fail for the expected reason (RED)**.
6. **Report** the tests added and what each asserts — then **STOP**. Do not implement
   (GREEN is a separate step, e.g. via `/implement`).

**Security hot spots get priority when choosing gaps.** Ownership and permission evaluation, SAS
URL issuance, and upload validation (content type, size caps, per-activity count) are the places
where an untested path means private media could reach the wrong caller. If a feature touches one
of those and the gap list is long, cover the security boundary first.

**Do not put Azure in the unit tier.** Anything that only proves the real blob implementation —
SAS generation, container existence, content-type round-tripping — belongs in the storage
integration tier, tagged so it can be excluded without credentials.
