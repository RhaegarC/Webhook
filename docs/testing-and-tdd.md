# Testing & TDD Standard

Status: **Draft** (2026-09-18)

A project-independent standard for how tests are written, which tier they belong to, and what the
TDD workflow guarantees. Referenced by the `tdd-implement` and `bug-fix` agents — the workflow
(RED → GREEN → refactor) runs on the tiers below.

Nothing here names a feature, a project, or a count. **Which behaviour a suite covers is status,
not standard**: it is written down in that project's own feature or sprint index. This document
says *how* a test is written and *where* it runs. A count restated in a standard goes stale in the
standard every time a feature adds a test.

## Scope

The standard applies to **backend/API code**. A UI built in a design tool and exported is not
test-first; the only hand-written frontend work is API integration, verified manually end-to-end.

Where the backend is a layered solution, each layer carries a sibling test project, and new tests
go in the project matching the layer they exercise.

## The tier model

**A test belongs to the tier that owns its claim kind.** A claim *decided by the request* — field
rules, request shape, what the service hands the repository — runs offline. A claim *about what the
store does* — column type, check constraint, soft-delete filter, audit stamp — runs against a real
engine, container-backed where it must be. No test project references another's, and no fake
database stands in for either.

| Tier | Claim kind | Tooling | Runs |
|---|---|---|---|
| **Unit** | Rules decided from input alone: validation, policy construction, pagination clamping, ownership and permission evaluation, and reflection assertions that a value has no second source or no other way in | Test framework only | Always — fast, offline |
| **Model** | The ORM's *model* and its *generated SQL* — keys, column types and lengths, soft-delete predicates read through a query-string build, a check constraint's SQL, and the absence of a relationship the docs might imply | Test framework + ORM | Always — offline, opens no connection |
| **Database** | The real engine: the migration set applies, the fluent bounds reached the schema catalogue, a duplicate key collides, the soft-delete filter executes, structured columns round-trip, a narrowing `ALTER` is refused, a check constraint refuses an out-of-set value, a default fills itself in, a row round-trips and is re-read in a fresh scope; and the database it creates outlives the run | Test framework + engine container | **Container-tagged** — skips when unreachable |
| **External service** | The real provider client: upload, content-type round-trip, a minted credential the server accepts, routing, a move | Test framework + provider SDK | **Container-tagged** — skips when unreachable |
| **Host** | The real pipeline booted in-process against unreachable connection strings — a missing setting stops startup and the message names the key, every registration in the composition root resolves, and a protected route answers 401 to an anonymous caller | Test framework + host factory | Always — no database, deliberately |

**A test double belongs to no tier.** Nothing in the suite stands in for a database or a storage
provider; see "No fakes for the store" below.

Use a **single container trait** as the tag, and treat the two obvious filters as different things.
`Category!=Container` is *not* "the offline run" if any container-tagged tier also runs bare by
falling back to an emulator or a local stand-in — that tier is container-tagged but never needs a
container, so the filter excludes it from a run it would have passed.

### What the offline tiers earn

The model tier compares the model to a snapshot and inspects generated SQL by building the
statement and stopping. That is two in-memory artefacts compared to each other: it proves what the
ORM *would* send, and it cannot prove the engine accepts it. It earns its place by catching a
mis-configuration in under a second, and by failing on a machine with no container at all.

Which is exactly why the database tier exists. A built-statement assertion and an executed one are
different claims, and the second is the one that catches a migration that never applied.

### The seam, stated rather than hidden

No tier drives request → repository → engine in one run, so a service's wiring is asserted against
a recording double while the store's behaviour is asserted through direct repository calls — and
neither test would catch the two being wired to each other wrongly. Where that seam exists, say so
plainly. Naming it is what stops it being mistaken for coverage.

### Claims that are withdrawn, not faked

When a behaviour does not exist, **withdraw the claim** rather than inventing a test for it. If the
model declares no foreign keys, there is nothing to cascade and cascade deletes are not a tier.
Inventing a test for an absent behaviour is how a suite acquires assertions that cannot fail.

Likewise, if an assertion holds only because of a framework ordering guarantee rather than your own
behaviour, it is not a test of your code. Say so and let a skip do the job instead.

## The container tier

### Skipping, not failing

Reaching nothing at the configured endpoint **skips**; a container that is simply not running is
not a bug in the code under test. Everything past the connect check is a **failure**, deliberately —
a migration that will not apply against a real engine is the exact bug this tier exists to find, and
degrading it to a skip would hide the one thing it was built for.

In .NET, `Skip.IfNot` works by throwing, which trips three things worth knowing before writing a
test here:

- It requires `[SkippableFact]` / `[SkippableTheory]`. Under a plain `[Fact]` the thrown
  `SkipException` is just an exception and the test **fails**.
- It must be called **outside** anything that catches. `Record.ExceptionAsync` and
  `Assert.ThrowsAsync` catch the `SkipException` too and hand it back as the exception under test,
  so the test reports a failure whose "expected" value is the skip message. Resolve the fixture's
  client or repository **before** the recorder.
- The fixtures should skip for you. Branching on an `IsAvailable` flag by hand is how a test ends up
  either not skipping or skipping for a reason it never states.

### One database, and one per test that needs its own

**Every container test works against the same database**, created once per run and migrated to head;
each test class gets a scope over it, not a database of its own. Isolation is therefore **by row**:
an assertion is scoped to the id it wrote, and nothing may assert on a table as a whole.

A transaction rolled back after each test is the obvious alternative and does not work: reading a
row back through the *same* context hits the ORM's change tracker rather than the database, and a
genuine second context needs a second connection inside one transaction, which needs a distributed
transaction coordinator that a Linux container usually does not have.

Row-level cleanup is the other candidate and is worse: it assumes hard delete. In a codebase whose
delete is *soft*, it means raw delete statements ordered by foreign key, extended by every future
feature, rotting silently.

A test that must drive a **schema of its own** — one that stops a migration early — cannot use the
shared database: that state must not leak into the next test, and a database cannot be returned to
an earlier migration once anything has taken it to head. Those tests get a **scratch database each**,
named with a unique suffix and dropped when the test that made it finishes.

### Retention

**The run's database is not dropped at teardown.** A failing container-backed assertion is diagnosed
by reading the rows behind it, so it is left standing — tables, rows and all. The next run drops it
before creating its own, so exactly one run's worth survives and the engine does not fill up with
databases.

A scratch database is the exception, and goes with the test that made it. That is what keeps the
run's leavings to the one database it means to leave.

Assert both halves: that the run's database outlives its disposal, and that a test's own does not.

**Inspect before tearing the container down.** A container without a volume takes the run's database
with it when removed. Stopping it, or leaving it running, keeps it.

### Migrations

**`Migrate()`, never `EnsureCreated()`.** `EnsureCreated` builds the schema from the model and
bypasses the migration assembly entirely, so a broken migration stays broken and the test still
passes — a test that cannot fail.

The migration set is applied **once per run**, by the same call that creates the database. Running
it from each fixture instead executes it once per test class, across classes running in parallel —
and the loser of a race over the migrations history table fails a create it did not need.

## External services

Route all vendor access through one abstraction, so the vendor is confined to a single file and the
contract names no vendor type. That is what makes the external-service tier swappable between an
emulator and the real thing.

Run the tier against a **local emulator by default**, which needs no credentials, and against the
real service when a connection string names one. Both reach the same code; only the account behind
it differs. That is what makes the tier runnable on any machine rather than only on a credentialed
CI branch.

Provisioning may set access levels or permissions on an emulator — but **only when the endpoint is
loopback**. A test run must never change the access level of a resource in a real account: that is a
deployment change with a security consequence, made silently, by a process whose whole job is to
observe.

## No fakes for the store

**There is no fake implementation of the repository or storage contract, and its absence is
deliberate.**

A fake implements the contract it is meant to be asserting, which means a test asserting "write then
read back" proves only that a dictionary tolerates a key. **A fake shadows the real implementation's
invariants while appearing to test them.** Anything provider-specific — credential issuance,
container existence, content-type round-tripping — is proven against a real instance or not at all.

What that costs is real and is stated rather than hidden: the RED → GREEN loop for store-facing code
is not instant, and the tier skips on a machine with no emulator. What it buys is that no store
assertion in the suite is an assertion about a fake.

The one sanctioned use of a double is a **purpose-built recording double**, defined inside the single
test that needs it, whose only job is to record whether it was called. Its claim is "the request was
rejected *before* any store operation" — a property that cannot be tested any other way. Such a
double records and does nothing else; it is not an implementation of the contract and not a
substitute for the container tier.

## TDD discipline

1. **RED** — write a failing test for the behaviour first; run it to confirm it fails *for the right
   reason*.
2. **GREEN** — minimal implementation to pass.
3. **Refactor** while green; run the full tier.

**Must be test-first (hot spots).** These are the places where a passing suite is the only evidence
the system is not quietly doing the wrong thing. The categories generalise to any project; the
specifics come from that project's security model:

- **Authorization and ownership evaluation** — the security boundary of the application: who may
  read, who may mutate, the administrative override, and anonymous denial everywhere except the
  deliberately public endpoints.
- **Credential issuance** — that an unauthenticated or unauthorized caller is rejected *before* any
  provider operation happens, and that the credential's lifetime is bounded and enforced
  server-side.
- **Input validation at the boundary** — allowlists, size caps, and per-owner count caps, each with
  a rejection test *at* the boundary value rather than only inside it.
- **Anything where a caller's string reaches a query, a path, or a command structurally** — where
  parameterization is not possible, the value is validated against an allowlist shape before use,
  and the rejection is asserted.

### When there is no behaviour to drive

A correct change does not always have runtime behaviour to test. Two further shapes are sanctioned,
so that such a change is not forced into a test that cannot fail:

- **`doc-assertion`** — the change protects a durable property of a *file*: "every command is listed
  in the README", "no permission entry names another repository". Write a test that reads the file
  and asserts the property, and confirm it goes red when the file is reverted. It is a real test
  with a real failure mode.
- **`verification-only`** — nothing can be asserted at all: a workflow file, a deployment step, a
  request line in an HTTP file. Record what was run and observed in a `Verification:` line, and say
  in the pull request why there is no test. **Do not invent a test that cannot fail** to make the
  change look finished.

A verified claim and a tested claim are different strengths of claim, and blurring them is worse
than either.

## Commands and configuration

```bash
<start the test dependencies>     # e.g. a test-specific compose file
<run the tests>                   # everything runnable on this machine
<run the tests> --filter "<ContainerTrait>"     # the container tiers alone
<run the tests> --filter "!<ContainerTrait>"    # the tiers touching no container
<stop the test dependencies>
```

Keep the **test** dependency file separate from the **local development** file, and do not treat one
as a substitute for the other. Starting the local stack changes nothing about the test run: with only
the stack up, the container tiers still **skip**, exactly as they do on a bare machine. The skip is
about the test database and the emulator, not about anything running in Docker.

**Test variables are named distinctly from the application's own.** This is load-bearing: a developer
with a working application setup has the application's connection variables in their shell already,
pointing at a real environment. A test tier that read those would have the test run create and drop
databases in production. Give the test tier its own variable names, and have the test fixture compose
its connection string from those.

`.env` is gitignored; `.env.example` is tracked and holds the shape without the secret.

**A skip is reported in the run summary.** It is a skip, not a silent exclusion, so a tier cannot be
forgotten by vanishing from the output.
