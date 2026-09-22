# 04 — Health and Observability

Status: **Not started** · [00-mission-1-sprint.md](00-mission-1-sprint.md)
Source: [PRD](../PRD.md) — FR-18…FR-22, §6.3, §4.4, Decision #12.

## Summary

Two endpoints that answer "is this thing actually working?" — and the distinction between them is the
point.

`GET /health` is anonymous and says nothing but whether the service is alive. `GET /health/status` is
API-key protected and reports the numbers that tell you whether it is *healthy*: how many rows exist,
how large the database and its WAL file have grown, and what period the data covers (Decision #12).

The genuinely important requirement is what `/health` **checks**. A `SELECT 1` is nearly useless here:
if the Docker volume ends up root-owned while the container runs non-root, reading an existing database
file can succeed while every write fails. A read-only health check goes green on a completely broken
deployment, and the first real webhook is what discovers the problem (PRD §6.3). **So `/health` checks
writability, not just readability.**

## Story

As the operator, I want a health check that fails when the service cannot actually do its job, so that a
green light means something.

## Dependencies

- [01-foundation](01-foundation.md) — the `DbContext`, the WAL configuration whose sidecar files this
  feature reports, and the startup write probe that this feature's runtime check complements

## Acceptance criteria

- [ ] `GET /health` is reachable **anonymously** and returns `200` when the service is healthy (FR-18)
- [ ] `GET /health` discloses **no** data detail — no row counts, no file sizes, no date ranges (FR-18).
      It is reachable through the public tunnel, so anything it returns is public
- [ ] `GET /health` verifies the database is **readable** (FR-19)
- [ ] `GET /health` verifies the data directory is **writable** (FR-19). This is the requirement that makes
      the endpoint worth having; see the summary above
- [ ] `GET /health` returns a non-2xx status when either check fails, so the container healthcheck and any
      future monitor see it
- [ ] `GET /health/status` requires the API key and returns `401` without it (FR-20)
- [ ] `GET /health/status` reports:
  - [ ] total row count
  - [ ] database file size in bytes
  - [ ] **WAL file size in bytes** — reported separately because WAL creates `webhook.db-wal` and
        `webhook.db-shm` beside the database, and all three are part of the data. A backup that copies only
        `.db` silently loses recent writes (PRD §6.2)
  - [ ] oldest and newest `CreatedOn`
- [ ] `/health/status` reports the **actual** file size from the filesystem, not a row-count estimate — the
      number exists to make disk growth visible (PRD §4.4)
- [ ] `/health/status` stays fast as the table grows. Row count is a table scan on SQLite unless it is
      answered by an index; the count query is examined (via `ToQueryString()` or equivalent) and is not
      permitted to degrade into a full scan of `RawPayload` rows
- [ ] `/health` remains reachable with **no** API key after [03](03-query-api.md)'s key middleware is
      applied, proving that middleware is scoped rather than global
- [ ] Both endpoints answer within the container `HEALTHCHECK` interval, including on a database with a
      large payload volume

## Tests (TDD)

- Integration (`Webhook.Api.Test`) — **hot spot (the writability check)**:
  - **`GET /health` fails when the data directory is not writable.** Constructed by pointing the app at a
    read-only directory and asserting a non-2xx status. This is the test that proves the deployment hazard
    in PRD §6.3 is actually caught rather than merely described
  - `GET /health` fails when the database file is unreadable
  - `GET /health` succeeds on a healthy database
  - **the `/health` response body contains no row count and no byte figures**, asserted on the serialized
    body
  - `GET /health` with no API key → `200` (anonymous access is the requirement, not an accident)
  - `GET /health/status` with no key → `401`; with the key → `200`
  - `/health/status` on an empty database reports count `0` and null/absent date range rather than throwing
  - `/health/status` after ingesting N deliveries reports exactly N, and a non-zero database size
  - the reported `oldest`/`newest` match the inserted rows' `CreatedOn` values

## Notes / non-goals

- **No alerting.** `/health/status` reports the size; nothing watches it. Disk growth is a Mission 2
  concern (PRD §11).
- **No metrics endpoint, no Prometheus, no OpenTelemetry.** Per-delivery structured logging
  ([02](02-ingest-endpoint.md), FR-21/22) plus these two endpoints is the whole observability story for
  Mission 1.
- **No log shipping or structured sink** (Serilog is a Mission 2 candidate).
- **No retention.** Nothing prunes the table, so `/health/status` will only ever grow (Decision #11).
