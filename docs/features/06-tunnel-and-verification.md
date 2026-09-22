# 06 — Tunnel and Verification

Status: **Not started** · [00-mission-1-sprint.md](00-mission-1-sprint.md)
Source: [PRD](../PRD.md) — §6.5, §8, §11; Decisions #9, #2.

## Summary

The moment the service starts doing its job. Until this feature lands, everything built in 01–05 listens
on `localhost` and has never received a single real event.

Two things happen here: a **named Cloudflare tunnel** gives the service a stable public hostname, and
GitHub is pointed at it. Then the whole thing is verified end-to-end against real deliveries.

The stability requirement is the whole reason for choosing a named tunnel. The webhook URL is configured
**once, in GitHub's UI**. A quick tunnel's random `*.trycloudflare.com` hostname rotates on every restart,
which would mean re-pasting the URL into GitHub after every restart — and while it is stale, GitHub's
deliveries fail. **A stable hostname is a requirement, not a preference** (PRD §6.5).

This feature is also the only place the two open questions in PRD §11 can be answered, because both are
observations about how GitHub actually behaves rather than things the code can assert.

## Story

As the operator, I want GitHub's events to arrive at a stable public address and be recorded, so that the
service is actually capturing rather than merely running.

## Dependencies

- [05-containerization](05-containerization.md) — the `cloudflared` service and the `CF_TUNNEL_TOKEN`
  configuration slot this feature fills

## Acceptance criteria — provisioning

- [ ] A Cloudflare-managed domain exists and the operator can create DNS records on it
- [ ] A **named** tunnel is created: `cloudflared tunnel login` → `tunnel create webhook` →
      `tunnel route dns webhook webhook.<domain>` (PRD §6.5)
- [ ] The tunnel token is placed in `config/.env` as `CF_TUNNEL_TOKEN` and is **not** committed
- [ ] The hostname is **stable across restarts** — verified by restarting the stack and confirming the same
      URL still works. A rotating hostname means this feature is not done
- [ ] The public URL reaches the API: `https://webhook.<domain>/health` returns `200`
- [ ] TLS is terminated at Cloudflare and the app correctly sees the original scheme, confirming the
      forwarded-headers handling from [01](01-foundation.md) works. **A `307` redirect here means it does
      not** — that is the failure mode PRD §6.4 describes, and it presents as a silent non-delivery
- [ ] The API is **not** reachable directly by hostname or IP from outside; the tunnel is the only path (SEC-5)
- [ ] The query API requires its key over the public URL, not merely on localhost

## Acceptance criteria — GitHub configuration

- [ ] A webhook is configured on the repository with payload URL
      `https://webhook.<domain>/api/webhooks/github`, content type `application/json`, and the shared secret
- [ ] GitHub's `ping` on save is **received and stored** — this is the wiring proof, and
      [02](02-ingest-endpoint.md) deliberately stores `ping` rather than dropping it
- [ ] A real event (merge a PR, leave a comment) is received and stored
- [ ] The delivery appears in GitHub's own webhook delivery log with a **2xx** response and a plausible
      response time

## Acceptance criteria — PRD §8 acceptance

All seven, restated here as the definition of done for Mission 1:

- [ ] `dotnet test` passes every case in [PRD §7](../PRD.md#7-testing-requirements)
- [ ] `docker compose up --build` starts both containers; logs show the resolved database path and a
      successful startup write probe
- [ ] `GET /health` returns `200` anonymously with no data detail; `GET /health/status` returns row count
      and file sizes with a valid key and `401` without
- [ ] A locally-signed POST stores one row; repeating it with the same `X-GitHub-Delivery` stores nothing
      further and still returns `200`; a corrupted signature returns `401` and stores nothing
- [ ] The GitHub webhook is configured against the public hostname; the `ping` is stored; a real PR merge is
      subsequently received and stored
- [ ] `docker compose down` then `up` (without `-v`) preserves all rows
- [ ] `docker compose down -v` then `up` starts cleanly on a fresh volume — the ownership hazard in
      [05](05-containerization.md) does not bite on the first attempt

## Verification

Ordered roughly cheapest-first, so a failure is localised early:

1. `dotnet test` — all PRD §7 cases
2. `docker compose up --build`; both containers healthy
3. Local smoke test with a hand-computed signature, bypassing the tunnel entirely:
   ```
   BODY='{"action":"closed","repository":{"id":1,"full_name":"me/repo"},"sender":{"login":"me"}}'
   SIG=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')
   curl -i -X POST http://localhost:8080/api/webhooks/github \
     -H "X-GitHub-Event: pull_request" -H "X-GitHub-Delivery: test-1" \
     -H "X-Hub-Signature-256: sha256=$SIG" -d "$BODY"
   ```
   Expect `200`; rerun unchanged → `200` and still one row; corrupt `$SIG` → `401` and no row
4. `GET /api/deliveries` → one row, no payload; `GET /api/deliveries/{id}` → payload included
5. Through the tunnel: `https://webhook.<domain>/health` → `200`, no `307`
6. Configure GitHub; confirm the `ping` lands as a stored row
7. Merge a real PR; confirm the `pull_request` event lands
8. Persistence: `down` / `up` (no `-v`) → rows survive
9. Fresh volume: `down -v` / `up` → clean start, no ownership failure

## Notes / non-goals

- **No tunnel redundancy, no load balancing, no failover.** One tunnel, one host.
- **No Cloudflare Access or WAF** in front of the query API. The static API key is the access control; a
  WAF is a Mission 2 candidate (PRD §10).
- **No uptime monitoring.** Nothing watches `/health` from outside.
- **No backfill.** Events delivered before this feature landed are gone — GitHub does not queue deliveries
  and does not retain a replayable history on your behalf. If capturing early matters, run the tunnel
  against a running [01](01-foundation.md)–[02](02-ingest-endpoint.md) before containerization; the tunnel
  is independent of it, only the target hostname changes (see the sequencing note in
  [00-mission-1-sprint.md](00-mission-1-sprint.md)).

## Open questions this feature retires

Both were unverifiable at design time and are recorded in [PRD §11](../PRD.md#11-open-items-and-risks).
Neither changes the design — the deduplication approach is correct either way — but both should be
observed and written down here:

- **Does GitHub retry a failed delivery?** And with what backoff? Observed by failing a delivery
  deliberately (e.g. a `401` from a temporarily wrong secret) and watching GitHub's delivery log.
- **Does manual redelivery from the GitHub UI preserve the original `X-GitHub-Delivery` GUID?** Observed by
  redelivering an earlier event and checking whether the row count increases. A stable GUID deduplicates
  cleanly; an unstable one behaves as if no constraint existed — in which case the answer is "redelivery
  creates a second row", which is worth knowing rather than assuming.

Also worth recording once real traffic exists: the observed payload size distribution, which is what makes
the growth projection in [PRD §4.4](../PRD.md#44-retention) a measurement rather than an estimate.
