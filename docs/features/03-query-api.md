# 03 — Query API

Status: **Not started** · [00-mission-1-sprint.md](00-mission-1-sprint.md)
Source: [PRD](../PRD.md) — FR-12…FR-17, SEC-2, Decisions #2, #6, #7.

## Summary

The read surface: what was received, newest first, filtered however you need it.

Two decisions shape it. **The list omits the raw payload** — payloads are 5–50 KB each, so inlining a
hundred of them per page is megabytes of data the caller rarely wants; the body is available on the
detail read instead. And **event-specific fields are queried by JSON path rather than by dedicated
columns**, so a new event type costs no migration and no deploy (Decision #6). This endpoint is reachable
through the same public tunnel as the ingest endpoint, which is why it carries an API key (SEC-2).

## Story

As the operator, I want to look up what GitHub sent — by time, event type, repository, or a field inside
a specific event — so that I can see what happened without opening the database by hand.

## Dependencies

- [01-foundation](01-foundation.md) — schema, `DbContext`, indexes
- [02-ingest-endpoint](02-ingest-endpoint.md) — the rows this reads. The feature is testable with rows
  inserted directly, but it has nothing to answer until 02 exists

## Acceptance criteria

- [ ] `GET /api/deliveries` returns a paged list ordered by `CreatedOn` **descending** (FR-12)
- [ ] Filterable by event type, repository, and a `CreatedOn` range (FR-13)
- [ ] Filterable on **event-specific fields via JSON path** (FR-14) — e.g. `$.pull_request.number`
      answered by `json_extract(RawPayload, …)` against SQLite's built-in JSON1 support. Onboarding a new
      event type must require **no schema change and no deploy**
- [ ] The list response **omits `RawPayload`** (FR-15)
- [ ] `pageSize` is **clamped to a maximum of 100** (FR-16) — clamped, not rejected. An over-large request
      returns 100 rows rather than an error
- [ ] `GET /api/deliveries/{id}` returns one delivery **including** its raw payload (FR-17)
- [ ] Both endpoints require a static API key in a request header; a missing or wrong key returns `401` (SEC-2)
- [ ] The API key check is middleware, applied to the read surface only — the ingest endpoint is
      authenticated by signature and the anonymous `/health` in [04](04-health-and-observability.md) must
      stay anonymous
- [ ] A request for an id that does not exist returns `404`, not `500` and not an empty `200`
- [ ] Pagination is **offset-based**, deliberately (Decision #7). Growth is a few rows an hour; cursor
      pagination is real complexity for a problem this service will not have
- [ ] A filter whose JSON path matches nothing returns an empty page, not an error — a JSON path pointing
      at a field an event type does not carry is a legitimate query with a legitimate empty answer
- [ ] An **invalid** JSON path (malformed expression) returns `400` with a message naming the offending
      path, rather than surfacing a raw SQLite error
- [ ] Query parameters are composed as **parameters**, never interpolated into SQL. The JSON path is the
      only piece that cannot be parameterized — it is validated against an allowlist shape before use,
      and this is the one place in the codebase where a caller's string reaches SQL structurally

## Tests (TDD)

- Integration (`Webhook.Api.Test`) — **hot spot (the JSON path reaching SQL structurally)**:
  - a JSON path that is not a well-formed expression → `400`, and **no SQL executed** — asserted with a
    recording double or by inspecting the generated command, so "rejected first" is a tested property
    rather than a code-reading claim
  - a path shaped like an injection attempt is rejected by the validator, not by luck
  - without an API key → `401`; with a wrong key → `401`; with the right key → `200`
  - `/health` remains reachable with **no** key, proving the middleware is scoped and not global
  - `pageSize=500` → returns at most **100** rows
  - `pageSize=0` and a negative `pageSize` → behave predictably rather than throwing
  - the list response contains **no** `RawPayload` field, asserted on the serialized body — not merely on
    the DTO's shape, which would not catch a property added later
  - the detail response **does** contain the raw payload, byte-identical to what was ingested
  - ordering is `CreatedOn` descending; rows inserted out of chronological order still come back newest-first
  - a `since`/`until` range excludes boundary rows as documented
  - filtering by `$.pull_request.number` returns exactly the matching deliveries
  - an unknown id → `404`

## Notes / non-goals

- **No writes.** The query API is read-only. Nothing can be edited or deleted through it.
- **No full-text search across payloads.** JSON-path filtering answers field-specific questions; free-text
  search over 50 KB bodies is a different feature with a different index, and nothing has asked for it.
- **No cursor pagination** (Decision #7).
- **No `{n}` "latest n records" endpoint.** It was proposed and rejected: it collides with
  `GET /api/deliveries/{id}` (both bind integers, making the route genuinely ambiguous — the framework
  either throws at startup or silently picks one), and it duplicates this list endpoint with
  `?pageSize=n`. See [backlog.md](backlog.md).
- **No response caching and no rate limiting.** The API key is the only access control. A WAF or Cloudflare
  Access in front of this endpoint is a Mission 2 candidate (PRD §10).
