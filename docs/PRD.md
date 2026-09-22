# PRD — GitHub Webhook Receiver

| | |
|---|---|
| **Status** | Draft — pending review |
| **Date** | 2026-09-22 |
| **Phase** | Phase 1 (capture only) |
| **Target branch** | `develop` |

---

## 1. Context

GitHub can POST a payload to an HTTP endpoint whenever something happens in a repository —
a pull request is merged, a comment is left, a branch is pushed. The payload describes the
event, the repository, and the actor. This is the *webhook* mechanism, and it is how GitHub
tells the outside world that something changed.

There is currently no endpoint to receive these payloads. Events that occur between now and
whenever a receiver exists are **not recoverable** — GitHub does not queue deliveries on your
behalf and does not retain a replayable history of them for you.

This document specifies a service that receives, verifies, and durably stores those payloads,
and exposes a read-only query API over the stored data.

### Why phase 1 is capture-only

A later phase will have the service *react* to events — notify a channel, trigger a build, sync
elsewhere. Phase 1 deliberately does none of that. The reason is not scope-cutting for its own
sake; it is that **capturing and reacting have different failure semantics**. A capture service
must never lose data. A reactor must never act on the same event twice. Building them together
produces a component that is bad at both.

The one design constraint this imposes on phase 1: **the raw payload is stored verbatim as the
source of truth.** Anything derived from a payload can be re-derived later; a payload that was
never stored cannot be recovered. The ingest path is therefore kept structured such that an
outbox/processor can be added in phase 2 without reworking phase 1.

---

## 2. Goals and non-goals

### Goals

- **G1** — Receive GitHub webhook deliveries reliably and respond within GitHub's timeout.
- **G2** — Reject unauthenticated deliveries; only payloads proven to come from GitHub are stored.
- **G3** — Store every accepted payload durably, surviving container restarts and redeploys.
- **G4** — Support duplicate deliveries without creating duplicate records.
- **G5** — Provide a query API for retrieving stored events by time, type, repository, and
  event-specific fields.
- **G6** — Run as a single Docker container plus a tunnel, brought up with one command.

### Non-goals (phase 1)

- **N1** — Reacting to events (notifications, builds, downstream sync) — phase 2.
- **N2** — A user interface. The query API is consumed directly (curl, a script, a future client).
- **N3** — Multi-tenant operation. This serves repositories owned by a single operator, behind
  one shared secret.
- **N4** — Retention/pruning policies. All events are retained.
- **N5** — CI/CD. The service is deployed manually to a local machine.
- **N6** — High availability, horizontal scaling, or clustering. Single instance by design.

---

## 3. Functional requirements

### 3.1 Ingest

| ID | Requirement |
|---|---|
| **FR-1** | The service SHALL expose `POST /api/webhooks/github` as the delivery endpoint. |
| **FR-2** | Every delivery SHALL be authenticated by verifying the `X-Hub-Signature-256` header as an HMAC-SHA256 of the **exact raw request body**, keyed by a configured shared secret. |
| **FR-3** | Signature comparison SHALL be timing-safe (`CryptographicOperations.FixedTimeEquals`). |
| **FR-4** | A delivery failing verification SHALL be rejected with `401` and SHALL NOT be persisted. The response body SHALL NOT disclose whether the header was absent, malformed, or incorrect. |
| **FR-5** | The payload SHALL be treated as **untrusted until verification succeeds**. No field from the body may be used for routing, secret selection, or storage before the signature is verified. |
| **FR-6** | Verification SHALL be implemented in middleware rather than in the controller action, so that no endpoint can omit it and the trusted/untrusted boundary exists in exactly one place. |
| **FR-7** | A verified delivery SHALL be stored with its raw body verbatim, alongside extracted index columns (§4). |
| **FR-8** | A delivery whose `X-GitHub-Delivery` value already exists SHALL NOT create a second record. |
| **FR-9** | Duplicate deliveries SHALL return `200`, not an error status. A duplicate is the expected outcome of a retry, not a failure. |
| **FR-10** | All GitHub event types SHALL be accepted, including `ping`. Unknown event types SHALL NOT be rejected. |
| **FR-11** | The endpoint SHALL respond within GitHub's delivery timeout under normal operation. |

### 3.2 Status code contract

Non-2xx statuses are returned **only when retrying would plausibly help**.

| Situation | Status |
|---|---|
| Signature invalid, or `X-Hub-Signature-256` absent | `401` |
| Signature valid, body is not parseable JSON | `400` |
| Valid delivery stored | `200` |
| Valid delivery, duplicate `DeliveryId` | `200` |
| Valid delivery, storage failed | `500` |

### 3.3 Query API

| ID | Requirement |
|---|---|
| **FR-12** | `GET /api/deliveries` SHALL return a paged list of stored deliveries, ordered by `CreatedOn` descending. |
| **FR-13** | The list SHALL support filtering by event type, repository, and a `CreatedOn` range. |
| **FR-14** | The list SHALL support filtering on **event-specific fields via JSON path** (e.g. `$.pull_request.number`), so that onboarding a new event type requires no schema change. |
| **FR-15** | The list response SHALL **omit** the raw payload. Payloads are 5–50 KB; inlining 100 per page is megabytes of data the caller rarely wants. |
| **FR-16** | `pageSize` SHALL be clamped to a maximum of 100. |
| **FR-17** | `GET /api/deliveries/{id}` SHALL return a single delivery **including** its raw payload. |

### 3.4 Health and observability

| ID | Requirement |
|---|---|
| **FR-18** | `GET /health` SHALL be anonymously reachable and SHALL NOT disclose row counts, file sizes, or other data detail. |
| **FR-19** | `GET /health` SHALL verify the database is readable **and** that the data directory is writable. A read-only check is insufficient — see §6.3. |
| **FR-20** | `GET /health/status` SHALL be API-key protected and SHALL report row count, database file size, WAL file size, and the oldest/newest `CreatedOn`. |
| **FR-21** | Each delivery SHALL emit one structured log line recording delivery ID, event type, repository, and outcome (`stored` / `duplicate` / `rejected`). |
| **FR-22** | Signature verification failures SHALL be logged at Warning level. Repeated failures are the primary signal of probing. |

---

## 4. Data requirements

### 4.1 Schema

A single table, `WebhookDeliveries`:

| Column | Type | Notes |
|---|---|---|
| `Id` | `long` | Surrogate primary key (SQLite rowid) |
| `DeliveryId` | `string` | `X-GitHub-Delivery`; **unique index** |
| `EventType` | `string` | `X-GitHub-Event` — `pull_request`, `push`, … |
| `Action` | `string?` | `payload.action`; **nullable** — absent for `push`, `ping` |
| `RepositoryFullName` | `string` | `payload.repository.full_name`; indexed |
| `RepositoryId` | `long?` | `payload.repository.id` — stable across renames |
| `Sender` | `string?` | `payload.sender.login` |
| `RawPayload` | `string` | Verbatim request body |
| `CreatedOn` | `DateTimeOffset` | Insert time, **UTC**; indexed |

### 4.2 Rationale for non-obvious choices

- **Surrogate integer PK, not `DeliveryId` as PK.** A GUID primary key in SQLite gives every index
  entry and row lookup a random 36-byte key with poor locality. An integer PK is the rowid (free),
  and the unique index on `DeliveryId` enforces deduplication beside it.
- **`Action` is nullable.** `push` and `ping` have no `action`. A non-null constraint would require
  a magic `"none"` string that every query must special-case forever.
- **No per-event columns.** Twenty event types would mean twenty spurious nullable columns. SQLite
  ships JSON1, so `json_extract(RawPayload, '$.pull_request.number')` answers event-specific queries
  with no migration. New event types cost zero schema work (FR-14).
- **`RepositoryId` alongside `RepositoryFullName`.** Repositories are renamed and transferred;
  `full_name` is mutable, `id` is not. Without the id, events cannot be reliably joined across a
  rename.
- **`CreatedOn` is `DateTimeOffset` in UTC.** SQLite has no native `DateTimeOffset`; EF Core stores
  it as ISO-8601 `TEXT`, which sorts correctly **only under a uniform offset**. All writes are UTC.

### 4.3 Rationale for `CreatedOn` naming

`CreatedOn` with type `DateTimeOffset` matches the existing convention in the TrailBlaze codebase
(47 usages, vs 2 for `CreatedAt`). An earlier proposal had a separate `ReceivedAt` column; it was
rejected because the ingest path is synchronous, so "request arrived" and "row created" are the same
instant — two columns holding the same value is a latent bug awaiting a wrong filter.

### 4.4 Retention

No pruning in phase 1 (N4). Projected growth is ~150 MB/year at 20 events/day and ~1.5 GB/year at
200 events/day. SQLite handles multi-GB databases without degradation, so this is a question of
wanting the data, not of performance.

Adding retention later requires `DELETE` **plus** `VACUUM` — deleting rows does not shrink a SQLite
file, and without `VACUUM` the file remains at its high-water mark. This is called out because
`CreatedOn` is indexed specifically so retention stays cheap to add (FR-20 reports the size).

---

## 5. Security requirements

| ID | Requirement |
|---|---|
| **SEC-1** | The webhook secret SHALL be supplied via environment configuration, never committed. |
| **SEC-2** | The query API SHALL require a static API key in a request header. |
| **SEC-3** | The service SHALL **fail to start** if the webhook secret, API key, or database path is missing or blank. |
| **SEC-4** | The API SHALL not be reachable from the local network; its published port SHALL bind to loopback only. |
| **SEC-5** | Public reachability SHALL be provided only through the Cloudflare tunnel. |
| **SEC-6** | Secrets SHALL be generated with a CSPRNG (`openssl rand -hex 32`) and SHALL be distinct per purpose. |

### 5.1 Why SEC-3 exists (the fail-open trap)

The idiomatic configuration defaulting pattern is `${WEBHOOK_SECRET:-}`, which yields an **empty
string**, not null. An implementation that reads that value and computes an HMAC will find that
`Encoding.UTF8.GetBytes("")` succeeds — the HMAC is computed over a known-empty key. An attacker who
guesses "the secret is empty" can then forge signatures that verify successfully. The check passes
and fails **open**.

A container that refuses to start is a 30-second fix. A container that silently accepts forged
webhooks is a security hole with no symptom. Hence: validate at startup and throw.

---

## 6. Operational requirements

### 6.1 Deployment topology

Two containers, brought up by a single `docker compose up`:

| Container | Role |
|---|---|
| `webhook-api` | The ASP.NET Core service; SQLite embedded in-process |
| `cloudflared` | Cloudflare tunnel agent, dialing out to the edge |

The API's port is published to `127.0.0.1` only (SEC-4). `cloudflared` reaches the API over the
compose network, not through the published port.

### 6.2 Database embedding and persistence

SQLite is an in-process library, not a server — there is no database container, no port, and no
separate connection to establish. The API process opens the `.db` file directly.

The database file lives on a **named Docker volume** mounted at `/data`, so data survives
`docker compose down` and image rebuilds.

**WAL mode is enabled** (`journal_mode=WAL`), along with `synchronous=NORMAL`, `foreign_keys=ON`, and
a `busy_timeout`. Under WAL, writers do not block readers and readers do not block writers, which
removes a class of `database is locked` errors that are difficult to diagnose inside a container.

No background writer thread, queue, or channel is used. A local `INSERT` is sub-millisecond, and
introducing async machinery to avoid it would add a drop-on-`docker stop` failure mode in exchange
for no benefit.

> **Consequence worth knowing:** WAL creates `webhook.db-wal` and `webhook.db-shm` beside the
> database. All three files are part of the data. A backup that copies only `.db` silently loses
> recent writes.

### 6.3 The non-root / volume ownership hazard

.NET base images ship a non-root `app` user (UID 1654), and a **named Docker volume is created owned
by `root`**. Run non-root against a root-owned volume and the service dies with `SQLITE_CANTOPEN` —
or, worse, starts fine and fails on its first write.

Docker initializes a *fresh* named volume by copying ownership and contents from the image. So a
`/data` directory owned by `app` in the Dockerfile yields a correctly-owned volume. **But that copy
happens only once** — an existing volume is never re-initialized. If the container ever starts as
root, that volume is root-owned permanently, and no image rebuild repairs it; only `docker volume rm`
does.

Two mitigations are required:

1. **Startup write probe.** Attempt a real write to `/data` during initialization and throw on
   failure. This converts a mysterious runtime death into an immediate, loud, unambiguous failure.
2. **`/health` checks writability, not just readability** (FR-19). A read of an existing database can
   succeed while every write fails, so a read-only health check reports green on a broken deployment
   and the first real webhook is what discovers the problem.

### 6.4 HTTPS and reverse-proxy handling

The ASP.NET Core template enables `UseHttpsRedirection()`. **This must be removed.** Cloudflare
terminates TLS at its edge and forwards **plain HTTP** to the origin. The middleware sees a non-HTTPS
scheme and issues a `307` redirect; GitHub does not follow it usefully, and the webhook silently
never arrives.

Correct handling is `UseForwardedHeaders` honoring `X-Forwarded-Proto` and `X-Forwarded-For`, so the
application correctly understands it is behind TLS. This is registered first in the pipeline.

### 6.5 Tunnel

A **named Cloudflare tunnel** provides a stable hostname on the operator's own domain.

A quick tunnel (random `*.trycloudflare.com` hostname) is explicitly rejected: the URL is configured
once, in GitHub's UI. A hostname that rotates on every restart would require re-configuring GitHub
after every restart, and deliveries would fail in the interim. **A stable hostname is a requirement,
not a preference.**

Tunnel provisioning is one-time and manual: `cloudflared tunnel login` → `tunnel create` →
`tunnel route dns`. The resulting token is supplied to the `cloudflared` container via
`config/.env`.

### 6.6 Configuration and secrets

- `config/.env` holds real values and is gitignored.
- `config/.env.example` is committed and contains placeholders only, with a comment documenting
  `openssl rand -hex 32` as the generation method.
- Configuration keys reach the app via environment variables using the `__` separator
  (`Webhook__Secret` → `Webhook:Secret`).
- Local development outside Docker uses `dotnet user-secrets` — **not** `appsettings.Development.json`,
  which is committed and would place the webhook secret in git history permanently.

---

## 7. Testing requirements

Integration tests via `WebApplicationFactory`, mirroring the existing `TrailBlazeApiFactory` pattern.
Each test uses its own temporary SQLite **file**, not `:memory:` — in-memory SQLite requires holding
a connection open for the database's lifetime and sidesteps the WAL configuration under test, so it
would exercise a different configuration than the one shipped.

| Test | Asserts |
|---|---|
| Valid signature | `200`, one row stored |
| Bad signature | `401`, **zero** rows stored |
| Missing `X-Hub-Signature-256` | `401` |
| Duplicate `DeliveryId` | `200`, still **one** row |
| Stored payload byte-exact | `RawPayload` equals the raw body sent |
| Query API without key | `401` |
| Query API with key | `200` |
| `pageSize=500` | clamped to 100 |
| List endpoint | omits `RawPayload` |
| Detail endpoint | includes `RawPayload` |
| Blank `Webhook:Secret` at startup | host **throws** |

"Bad signature stores zero rows" is the highest-value assertion in the set. Verifying the signature,
returning `401`, and *still persisting the row* is a real defect — it passes every manual test and is
only caught by asserting on database state, not just on the status code.

---

## 8. Acceptance criteria

Phase 1 is complete when:

1. `dotnet test` passes all cases in §7.
2. `docker compose up --build` starts both containers; logs show the resolved database path and a
   successful startup write probe.
3. `GET /health` returns `200` anonymously with no data detail; `GET /health/status` returns row count
   and file sizes with a valid API key and `401` without.
4. A locally-signed POST to `POST /api/webhooks/github` stores one row; repeating it with the same
   `X-GitHub-Delivery` stores nothing further and still returns `200`; a corrupted signature returns
   `401` and stores nothing.
5. The GitHub webhook is configured against the public tunnel hostname; GitHub's `ping` on save is
   received and stored; a real PR merge is subsequently received and stored.
6. `docker compose down` followed by `up` (without `-v`) preserves all rows.
7. `docker compose down -v` followed by `up` starts cleanly on a fresh volume — the ownership hazard
   in §6.3 does not bite on the first attempt.

---

## 9. Decisions log

Decisions reached during design review, with the reasoning that produced them.

| # | Decision | Rationale |
|---|---|---|
| 1 | Phase 1 capture-only; raw payload is the source of truth | Derived data is recomputable; unrecorded payloads are lost forever |
| 2 | Multi-repo, one shared secret; `RepositoryFullName` indexed | Adding a repository becomes a GitHub-side change, not a deployment. Per-repo secrets would require parsing an unverified body to choose a secret |
| 3 | HMAC verification in middleware, `401`, timing-safe compare | Cannot be forgotten on a future endpoint; one trusted/untrusted boundary |
| 4 | Unique index on `DeliveryId`, conflict → no-op, `200` | Converts duplicate deliveries into a non-event. `409` would pollute GitHub's delivery log, the one place to check for missing events |
| 5 | EF Core over Dapper/raw ADO | Schema will evolve (phase 2 outbox); migrations make that a command rather than hand-run DDL on a volume |
| 6 | Generic columns + JSON-path queries, no per-event columns | Avoids a migration per event type |
| 7 | Offset pagination capped at 100; list omits payloads; API key on query API; `ping` stored | Growth is slow enough that cursor pagination is unjustified complexity; `ping` is the proof the wiring works |
| 8 | Debian `aspnet:10.0` base, non-root, named volume | SQLite's native library on glibc is the low-surprise path; Alpine trades ~100 MB for musl and globalization friction |
| 9 | Named tunnel, `cloudflared` as a compose service, token-based | See §6.5 — quick tunnels cannot serve a URL configured once in GitHub |
| 10 | `config/.env` + committed `.env.example`; fail fast on blank secrets | See §5.1 — defaulting to empty fails open |
| 11 | No pruning in phase 1 | Irreversible; disk is cheap; `CreatedOn` indexed so retention stays cheap to add |
| 12 | `/health` anonymous, `/health/status` key-protected | Keeps an anonymous public endpoint from disclosing data volumes |
| 13 | `WebApplicationFactory` integration tests | Mirrors the existing TrailBlaze test pattern; catches defects that manual testing cannot |
| 14 | Single project, not the layered TrailBlaze split | One table and two endpoints; six projects would be ceremony |
| — | `{n}` "latest n records" endpoint — **rejected** | Collided with `GET /{id}` (both integer routes) and duplicated the list endpoint with `?pageSize=n` |

---

## 10. Out of scope (phase 2 candidates)

- **Reactor / outbox.** A processing-state column and an outbox table. The schema and JSON-path query
  design were chosen so this is additive.
- **Retention.** `DELETE` plus `VACUUM`; requires the `CreatedOn` index already present.
- **Structured logging sink** (e.g. Serilog) and log shipping.
- **Cloudflare Access / WAF** in front of the query API, replacing or supplementing the static key.
- **CI/CD.** No GitHub Actions workflow in phase 1; deployment is manual.
- **Multi-tenancy.** Per-repository or per-organization secrets (N3).

---

## 11. Open items and risks

| Item | Status |
|---|---|
| `cloudflare/cloudflared` image tag | **Unverified** — Docker Hub was unreachable from the design environment. Official image; confirm on first `compose up`. |
| GitHub auto-retry behaviour | GitHub's retry policy on failed deliveries, and whether manual redelivery preserves the original `X-GitHub-Delivery` GUID, were not verified against GitHub's documentation. The deduplication design is robust either way: a stable GUID deduplicates cleanly, an unstable one behaves as if no constraint existed. |
| Deployment host | The design assumes a single always-on machine on which Docker runs continuously. If the host is a laptop that sleeps, deliveries during sleep are missed. |
| Disk growth | Not monitored automatically. `/health/status` reports size, but nothing alerts on it (FR-20). |
