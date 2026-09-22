# 05 — Containerization

Status: **Not started** · [00-mission-1-sprint.md](00-mission-1-sprint.md)
Source: [PRD](../PRD.md) — §6.1, §6.2, §6.3, §6.6, SEC-1, SEC-4, SEC-6; Decisions #8, #9, #10.

## Summary

The service becomes a container with a volume that survives restarts, and secrets stop living on the
command line.

The feature carries one genuinely nasty hazard, and most of the acceptance criteria below exist because
of it: **a named Docker volume is created owned by `root`, while .NET base images ship a non-root `app`
user.** Do both naively and the container starts, fails to open the database, and dies — or, worse, starts
fine and fails on its first write.

The fix is non-obvious. Docker initializes a *fresh* named volume by copying ownership and contents from
the image, so a `/data` directory owned by `app` in the Dockerfile yields a correctly-owned volume. **But
that copy happens only once** — an existing volume is never re-initialized. If the container ever starts
as root, that volume is root-owned permanently, and no image rebuild repairs it; only `docker volume rm`
does. The asymmetry is what makes this so confusing to debug.

## Story

As the operator, I want to bring the whole service up with one command and have its data survive restarts,
so that operating it is not a ceremony.

## Dependencies

- [01-foundation](01-foundation.md) — the fail-fast startup validation and write probe this feature's
  Dockerfile and compose file are what make necessary
- [02](02-ingest-endpoint.md), [03](03-query-api.md), [04](04-health-and-observability.md) — the endpoints
  the healthcheck and the smoke test exercise

## Acceptance criteria

- [ ] `src/Webhook.Api/Dockerfile`, multi-stage, following the existing FMS layout convention: Dockerfile
      in the project folder, build context at the repo root
- [ ] Build stage `mcr.microsoft.com/dotnet/sdk:10.0`, runtime stage `mcr.microsoft.com/dotnet/aspnet:10.0`
      (**Debian**, per Decision #8). Both tags verified present on MCR. Debian rather than Alpine: SQLite's
      native library on glibc is the low-surprise path, and Alpine trades ~100 MB for musl and ICU
      globalization friction. Not chiseled either — it has **no shell**, so you cannot `docker exec` in to
      inspect the database, which is exactly what this service will need
- [ ] `RUN mkdir -p /data && chown app:app /data` before `USER app` — the `chown` is what makes a fresh
      named volume inherit the correct ownership
- [ ] Container runs as the non-root `app` user (UID 1654)
- [ ] `ASPNETCORE_URLS=http://+:8080`; no HTTPS inside the container (TLS terminates at Cloudflare, PRD §6.4)
- [ ] `HEALTHCHECK` against `GET /health`, sized so it does not flap on a cold start
- [ ] `compose.yml` at the repo root, mirroring the FMS layout: top-level `name:`, `${VAR:-default}`
      overrides, explicit `container_name`
- [ ] Service `webhook-api` publishes its port to **`127.0.0.1` only**, never `0.0.0.0` (SEC-4). Public
      reachability comes solely through the tunnel (SEC-5)
- [ ] Service `cloudflared` using the official image, run with `tunnel --no-autoupdate run
      --token ${CF_TUNNEL_TOKEN}`, reaching the API at `http://webhook-api:8080` over the compose network
      rather than through the published port (Decision #9). Token-based, so no credentials JSON to mount
- [ ] Named volume `webhook-data` mounted at `/data`, declared under `volumes:`
- [ ] `config/.env.example` **committed**, containing placeholders only, with a comment documenting
      `openssl rand -hex 32` as the generation method
- [ ] `config/.env` **not** committed — already covered by the `.env` rule in
      [src/.gitignore](../../src/.gitignore), verified rather than assumed
- [ ] Configuration reaches the app via environment variables using the `__` separator
      (`Webhook__Secret` → `Webhook:Secret`), per PRD §6.6
- [ ] Secrets are distinct per purpose (SEC-6): the webhook secret, the API key, and the tunnel token are
      three separate values, none reused
- [ ] `docker compose up --build` starts **both** containers and the app comes up healthy
- [ ] Container logs show the resolved database path and a successful write probe on a clean start
      (this is [01](01-foundation.md)'s probe, and this is where it earns its place)

## Tests / verification

Containerization is verified by running it, not by unit tests. The checks below are the ones that
distinguish "it started" from "it works":

- [ ] `docker compose up --build` → both containers healthy; `docker compose ps` shows `healthy` not merely `running`
- [ ] `docker compose logs webhook-api` shows the resolved DB path and the write probe succeeding
- [ ] `curl -i http://localhost:8080/health` → `200`, anonymously
- [ ] **Persistence:** ingest at least one delivery, then `docker compose down` (no `-v`) followed by `up`
      → the rows are still there
- [ ] **Clean start on a fresh volume:** `docker compose down -v` then `up` → the service starts cleanly and
      writes on the first attempt. This is the ownership hazard; if it fails, the startup probe says so
      plainly instead of the failure appearing as a `500` on the first real webhook
- [ ] **The volume is not root-owned:** inspect the actual ownership on the volume and confirm it is `app`,
      not `root`. Asserting on the mechanism, not just on the symptom
- [ ] The published port is not reachable from another machine on the LAN (SEC-4)
- [ ] `docker compose down`/`up` leaves `webhook.db-wal` and `webhook.db-shm` on the volume alongside
      `webhook.db` — all three files are part of the data

## Notes / non-goals

- **No CI/CD.** No GitHub Actions workflow; images are built locally by `docker compose up --build`
  (PRD §10). The service is local-only by decision.
- **No image registry or tagging scheme.** Nothing is pushed anywhere.
- **No resource limits, no restart policy tuning, no orchestration** beyond Compose.
- **No `cloudflared` tunnel creation** — this feature supplies the container and the token's place in
  configuration; creating the tunnel is [06](06-tunnel-and-verification.md).
- **The `cloudflare/cloudflared` image tag is unverified.** Docker Hub was unreachable from the design
  environment. It is the official image; confirm the tag resolves on the first `compose up`, and note the
  result in the PRD if it differs (PRD §11).
