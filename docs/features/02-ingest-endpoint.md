# 02 — Ingest Endpoint

Status: **Not started** · [00-mission-1-sprint.md](00-mission-1-sprint.md)
Source: [PRD](../PRD.md) — FR-1…FR-11, §3.2, FR-21/FR-22, SEC-1, SEC-3, §5.1.

## Summary

The endpoint GitHub posts to. This is the security hot spot of Mission 1: it is the only part of the
service that accepts untrusted input from the public internet, and the only part whose failure mode is
"we silently stored something we should not have" or "we silently refused something we should have
kept".

The authenticated path is the easy part. **The difficult property is what happens on the failure path:
a rejected delivery must leave no trace.** Verifying a signature, returning `401`, and *still* having
persisted the row is a real defect — it passes every manual test, because the HTTP response looks
exactly right. It is only caught by asserting on database state.

## Story

As the operator, I want GitHub's events recorded exactly once and forgeries rejected outright, so that
the record is trustworthy and complete.

## Dependencies

- [01-foundation](01-foundation.md) — the `WebhookDelivery` schema, the `DbContext`, WAL configuration,
  and the fail-fast secret validation this feature's HMAC depends on

## Acceptance criteria

- [ ] `POST /api/webhooks/github` exists and accepts deliveries (FR-1)
- [ ] Every delivery is verified as an **HMAC-SHA256 of the exact raw request body**, keyed by the
      configured shared secret, compared against `X-Hub-Signature-256` (FR-2)
- [ ] Signature comparison uses `CryptographicOperations.FixedTimeEquals` — `string ==` leaks timing (FR-3)
- [ ] **The raw body is read as bytes and deserialized only *after* verification succeeds** (FR-5). The
      trap this avoids: taking `[FromBody] JsonDocument` lets model binding consume the request stream,
      after which the original bytes are unrecoverable — re-serializing produces different bytes and a
      guaranteed mismatch
- [ ] Verification lives in **middleware**, not in the controller action, so no future endpoint can omit
      it and the trusted/untrusted boundary exists in exactly one place (FR-6)
- [ ] A failing delivery returns `401` with an opaque body that does not disclose whether the header was
      absent, malformed, or merely incorrect (FR-4)
- [ ] **A rejected delivery is not persisted.** Asserted against database state, not the status code
- [ ] The verified payload is stored with its **raw body verbatim**, alongside the extracted index
      columns (FR-7). Byte-exact: what is stored equals what was sent, not a re-serialization
- [ ] A delivery whose `X-GitHub-Delivery` already exists creates **no second record** (FR-8)
- [ ] A duplicate returns `200`, **not** an error (FR-9). A duplicate is the expected outcome of a retry.
      `409` would be worse than wrong — it would mark the delivery failed in GitHub's delivery log, which
      is the one place to check whether events are going missing
- [ ] **All** event types are accepted, including `ping`; unknown types are **not** rejected (FR-10).
      `ping` is sent when the webhook is saved and is the proof the wiring works — dropping it would be
      perverse
- [ ] Status codes follow [PRD §3.2](../PRD.md#32-status-code-contract): `401` invalid signature / `400`
      unparseable body / `200` stored / `200` duplicate / `500` storage failure. The principle: **non-2xx
      only when retrying would plausibly help**
- [ ] One structured log line per delivery: delivery ID, event type, repository, outcome
      (`stored` / `duplicate` / `rejected`) (FR-21)
- [ ] Signature failures logged at **Warning** — repeated failures are the primary signal of probing (FR-22)
- [ ] The endpoint responds well within GitHub's delivery timeout under normal operation (FR-11)

## Tests (TDD)

- Integration (`Webhook.Api.Test`) — **hot spot (security — RED first)**:
  - **bad signature → `401` and zero rows stored.** The highest-value assertion in Mission 1
  - **missing `X-Hub-Signature-256` → `401`**
  - a signature valid for a *different* body → `401` (guards against binding the HMAC to the wrong bytes)
  - valid signature → `200` and exactly one row
  - **stored `RawPayload` is byte-identical to the body sent** — including formatting and key order, which
    is the assertion that fails if anything re-serializes
  - same `X-GitHub-Delivery` twice → both `200`, still exactly **one** row
  - `X-GitHub-Delivery` differs but the body is identical → **two** rows (dedup is on the delivery ID, not
    on content)
  - an unparseable body with a valid signature → `400`, zero rows
  - an event type never seen before → accepted, not rejected
  - a body of a few hundred KB is stored intact (guards against a body-size limit silently truncating)

## Notes / non-goals

- **No reactor.** Nothing is done with an event beyond storing it (Mission 1 is capture-only). The schema
  and the JSON-path query design in [03](03-query-api.md) exist so a reactor can be added in Mission 2
  without reworking this.
- **No per-repository secrets.** One shared secret, and the repository is read from the payload *after*
  verification. Per-repo secrets would require parsing an unverified body to decide which secret to check
  it against (Decision #2).
- **No queue or background processing.** Storage is synchronous and takes microseconds (PRD §6.2).
- **Body-size limits are left at the framework default.** GitHub caps payloads at 25 MB and the default
  permits 30 MB, so tightening is not required; the large-payload test above is a guard against a future
  change that makes it required.
- **Retry semantics are not implemented here.** The service does not ask GitHub to retry and cannot
  control whether GitHub does. The dedup design is correct whether or not retries happen: a stable
  `X-GitHub-Delivery` deduplicates cleanly, and an unstable one behaves as if no constraint existed
  (PRD §11).
