# Backlog

Ideas that are not yet features. The `/implement` and `/next` selection scans skip this file — only
numbered `docs/features/NN-name.md` files claim a priority slot. Promote an item by running
`/capture feature NN`, which stress-tests it into a real spec.

## Candidates

| Idea | Why it's not specced yet |
|---|---|
| **Reactor / outbox** | Mission 2's whole subject. Needs a processing-state column and an outbox table. The Mission 1 schema and the JSON-path query in [03](03-query-api.md) were chosen so this is additive rather than a rework — but nothing in Mission 1 consumes an event, so there is nothing yet to react *with*. |
| **Retention / pruning** | Deferred by Decision #11. Retaining everything is more valuable than saving disk while the data set is small (~150 MB/year at 20 events/day). Adding it later needs `DELETE` **plus** `VACUUM` — deleting rows does not shrink a SQLite file. `CreatedOn` is already indexed so the delete is cheap, and [04](04-health-and-observability.md) already reports the size the policy would act on. |
| **Alerting on disk growth** | [04](04-health-and-observability.md) reports database and WAL sizes but nothing watches them. The gap is real but small: the growth rate is measurable and slow, and an unused alert is worse than none. |
| **Cloudflare Access / WAF in front of the query API** | The static API key is the whole access control today (SEC-2). A WAF would add rate limiting and a second authentication layer, replacing or supplementing the key. Nothing has attacked the endpoint and it is not yet exposed. |
| **Structured logging sink (Serilog)** | Per-delivery structured logging already exists (FR-21) and goes to container logs, which is sufficient while the only reader is `docker logs`. Worth revisiting when a log is actually lost. |
| **CI/CD** | Explicitly out of scope (Decision #12 / PRD §10). The service is local-only, so a pipeline would have nothing to deploy to. Would also require deciding where images are pushed, which nothing currently does. |
| **Multi-tenancy / per-repository secrets** | Rejected for Mission 1 by Decision #2 in favour of one shared secret. A real requirement if the endpoint ever serves repositories the operator does not own — and note the design cost: per-repo secrets require parsing an **unverified** body to decide which secret to check it against, which is a genuine architectural change, not a config change. |
| **Free-text search across payloads** | [03](03-query-api.md) answers field-specific questions via JSON path. Searching inside 5–50 KB bodies is a different index and a different feature, and no one has asked for it. |
| **Backfill of pre-existing events** | Not possible. GitHub neither queues deliveries for an endpoint that does not exist nor retains a replayable history on your behalf. Listed here so it is not re-proposed: there is nothing to fetch from. |

## Rejected (do not re-propose without new information)

| Idea | Why it was rejected |
|---|---|
| **`GET /api/deliveries/{n}` — "latest n records"** | Collides with `GET /api/deliveries/{id}`: both bind an integer, so `GET /api/deliveries/5` is genuinely ambiguous — the framework either throws `AmbiguousMatchException` at startup or silently picks one, and silently picking one is worse. It also duplicates the list endpoint, which already sorts by `CreatedOn` descending and accepts `?pageSize=n`. A literal-segment form (`/api/deliveries/latest?count=n`) would avoid the collision but is a second name for one behaviour. |
| **Layered solution split** (`Model`/`Repository`/`Service`/`Interface`/`Api`, each with a sibling test project) | TrailBlaze's structure, and the wrong shape here (Decision #14). This is a middleware, two controllers, one `DbContext`, one entity, a Dockerfile and a compose file — roughly 400 lines. Six projects would mean four files to click through to follow one HTTP request, and a `.csproj` reference decision per new file. Revisit if the domain grows a second entity graph. |
| **A separate `ReceivedAt` column** | Proposed and dropped. The ingest path is synchronous, so "request arrived" and "row created" are the same instant; two columns holding one value is a latent bug awaiting a wrong filter. `CreatedOn` matches the TrailBlaze convention. If a second timestamp is ever needed, the meaningful distinction is `CreatedOn` (our clock) versus the event's own time from the payload — not arrival versus insert. |
| **Alpine or chiseled base images** | Considered against Debian (Decision #8). Alpine saves ~100 MB and buys musl plus ICU/globalization friction around SQLite's native library. Chiseled is the most secure of the three but ships **no shell**, so `docker exec` cannot be used to inspect the database — which is precisely what this service will need when something goes wrong on a volume. |
| **Quick Cloudflare tunnel** (`*.trycloudflare.com`) | Disqualified by topology, not by preference. The webhook URL is configured once in GitHub's UI; a hostname that rotates on every restart means re-pasting it after every restart, with deliveries failing in the interim. No amount of convenience is worth a webhook that only works until the next restart. |
| **Storing the payload as a normalized domain model** instead of verbatim | Considered in favour of raw-plus-extracted-columns. A normalized model forces a schema decision per event type before the events have been seen, and any field not modelled at ingest time is unrecoverable. Raw storage is strictly more information; normalization can be added later over the raw data, but not the reverse. |
