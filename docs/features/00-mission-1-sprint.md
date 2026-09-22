# Mission 1 — GitHub Webhook Receiver

Status: **Planning** — derived from [docs/PRD.md](../PRD.md) on 2026-09-22. The PRD stays the
living reference (decisions log, canonical schema, requirement IDs).

## Goal

Receive GitHub webhook deliveries, prove they genuinely came from GitHub, store them durably, and
answer questions about what was received. Capture only: reacting to events is Mission 2.

The one constraint that shapes Mission 1 is that **the raw payload is the source of truth**. Anything
derived can be recomputed later; a payload never stored is gone, because GitHub neither queues
deliveries nor retains a replayable history on your behalf.

## How to read these features (working model)

- **Single project.** Unlike TrailBlaze's layered `Model`/`Repository`/`Service`/`Api` split, this is
  one project — `src/Webhook.Api` — plus one sibling test project `src/Webhook.Api.Test`. The service
  is a middleware, two controllers, a `DbContext`, one entity, a Dockerfile and a compose file. Six
  projects for one table and two endpoints would be ceremony (PRD Decision #14).
- **Test-first.** RED → GREEN → refactor, via `WebApplicationFactory` integration tests. A test
  asserts on **database state**, not just status codes — "bad signature stores zero rows" is the
  assertion that proves the middleware ordering is right, and it is invisible to a status-code-only
  test. Test command: `dotnet test` (from `src/`).
- **Secrets never enter the repo.** `config/.env` holds real values and is gitignored; `config/.env.example`
  is committed with placeholders only.
- A feature that changes the schema updates the PRD data-model table in the same PR
  ([schema-change discipline](../PRD.md#4-data-requirements)).

### Sequencing note — the whole mission is undeployable until 05, and unreachable until 06

Features 01–04 build a service that only listens on `localhost` and has never been exposed. Feature
05 makes it a container with a persistent volume; feature 06 opens the tunnel and proves GitHub can
reach it end-to-end. Nothing is at risk in that order — an unexposed local service has no attack
surface — but it does mean **the webhook does not actually start capturing events until 06**, and
every day before that is a day of events GitHub delivered to nobody.

If capturing real events early matters more than finishing the build in order, run 06's tunnel
provisioning against a running 01–02 as soon as the ingest endpoint returns `200` locally. The
tunnel is independent of containerization; only the target hostname changes.

## Feature breakdown

Number = priority (lowest first = next to implement); file = `docs/features/NN-name.md`.

**This table is the only place a feature's status is written down.** The `Status` column owns
progress and the summary cell owns what the slice is; a feature's own file states its lifecycle and
nothing more.

| # | Feature (file) | Depends on | Summary — the slice | Status |
|---|---|---|---|---|
| 01 | [foundation](01-foundation.md) | — | Single project + test project; EF Core + SQLite with the `WebhookDelivery` entity and initial migration; WAL pragmas; fail-fast startup validation of secrets; removal of the template's `UseHttpsRedirection()` and `WeatherForecast`; forwarded-header handling; startup write probe to `/data` | not started |
| 02 | [ingest-endpoint](02-ingest-endpoint.md) | 01 | `POST /api/webhooks/github`: HMAC-SHA256 signature verification in middleware over the exact raw body, `401` on failure with nothing persisted, verbatim payload storage, `DeliveryId` deduplication, the status-code contract, and per-delivery structured logging | not started |
| 03 | [query-api](03-query-api.md) | 01, 02 | Static API key on the read surface; `GET /api/deliveries` paged newest-first with event/repo/date filters and JSON-path filtering on event-specific fields; `GET /api/deliveries/{id}` returning the raw payload. The list omits payloads; `pageSize` clamps to 100 | not started |
| 04 | [health](04-health-and-observability.md) | 01 | `GET /health` anonymous, verifying the database is readable **and** `/data` is writable; `GET /health/status` API-key protected, reporting row count, database and WAL file sizes, and the `CreatedOn` range | not started |
| 05 | [containerization](05-containerization.md) | 01–04 | Multi-stage Dockerfile on Debian `aspnet:10.0`, non-root `app` user with a correctly-owned `/data`; compose file with the API and `cloudflared`; named volume `webhook-data`; secrets via `config/.env`; port published to loopback only | not started |
| 06 | [tunnel-and-verification](06-tunnel-and-verification.md) | 05 | Named Cloudflare tunnel on the operator's domain, token supplied to the `cloudflared` service; GitHub webhook configured against the public hostname; the seven PRD acceptance criteria verified end-to-end, including persistence across `docker compose down` and a clean start on a fresh volume | not started |

## Mission-level risks

Carried from [PRD §11](../PRD.md#11-open-items-and-risks); each is owned by the feature that can
retire it.

| Risk | Owner |
|---|---|
| `cloudflare/cloudflared` image tag unverified — Docker Hub was unreachable from the design environment | 05 |
| GitHub's retry policy, and whether manual redelivery preserves the `X-GitHub-Delivery` GUID, not verified against GitHub's docs. The dedup design is robust either way | 02 (design) / 06 (observed in practice) |
| The design assumes an always-on host. If it is a laptop that sleeps, deliveries during sleep are missed and unrecoverable | 06 |
| Disk growth is reported by `/health/status` but nothing alerts on it | Mission 2 |
