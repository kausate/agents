# Error handling and anti-patterns

## Status codes returned by the public API

| Code | Meaning | What to do |
|---|---|---|
| `200` | Success — for async endpoints, check `status` in body | Branch on `status` |
| `201` | Created (monitor / webhook) | Persist returned id |
| `204` | Deleted | Resource is gone |
| `400` | Invalid input (malformed `kausateId`, invalid cron, unsupported jurisdiction) | Validate against the OpenAPI spec; check `detail` |
| `401` | Missing / invalid `X-API-Key` | Verify env var; rotate if leaked |
| `402` | Payment required (credits exhausted) | Top up; check `/v2/analytics/summary` |
| `403` | Forbidden (insufficient permissions, or feature not enabled for org) | Contact sales for premium features |
| `404` | Company / order / monitor / webhook not found | Verify the id; for orders, dedup may have returned an existing one |
| `408` | Sync request timed out (300 s hard limit) | Switch to async + webhooks |
| `422` | Pydantic validation failure (type mismatch, missing required field, body field that should be a header — e.g. `customerId` in body instead of `X-Customer-Id`) | Check the JSON-schema error in `detail` |
| `429` | Rate limited (gateway-level; not surfaced via headers) | Back off; rate limits are per-org |
| `500/502/503/504` | Server / registry-side error | Retry with exponential backoff; do not retry 4xx |

Error response shape:

```json
{ "detail": "human-readable error or pydantic error list" }
```

**Always retry 5xx with backoff. Never retry 4xx automatically** — they're caused by your request and won't change without a fix. Log every `orderId` you place and every webhook delivery for tracing.

## Anti-patterns — never generate this code

- **Sync mode (`/sync` endpoints) in production.** Hard 300 s timeout. Government registries are unreliable; you'll see HTTP 408 even when the data would eventually be available. Use sync only for tests and prototypes.
- **Polling faster than every 5 s.** It doesn't deliver results faster (registry latency dominates) and burns rate budget.
- **Treating `completed` as the only outcome.** Every closed status (`failed`, `canceled`, `terminated`, `timedOut`) needs handling.
- **Hardcoding the API base URL across environments.** Use `KAUSATE_API_BASE_URL` so dev / staging / prod can be swapped via env.
- **Storing the API key in source.** Read from env. Never commit `.env`.
- **Skipping `Kausate-Version`.** You will get bitten when the latest version moves on.
- **Doing real work inside the webhook handler.** Acknowledge in 2xx fast and queue.
- **Assuming a webhook won't be redelivered.** It will, up to 50 times. Be idempotent on `orderId`.
- **Mixing path styles for different `Kausate-Version`s.** With `2026-05-01` use `POST /v2/companies/report` (`kausateId` in body). With `2025-04-01` use `POST /v2/companies/{kausateId}/report`. Don't mix.
- **Putting `customerId` in the request body.** It's `X-Customer-Id` header. Body submission returns 422.
- **Calling `prefill` (or any data endpoint) with `companyNumber` / `jurisdictionCode` instead of `kausateId`.** Returns 422. Translate native registry IDs to `kausateId` via `search` first, then cache the mapping.
- **Sending `customerReference` longer than 150 chars or with non URL-safe characters.** Will 422.
- **Subscribing a webhook *after* placing the order and expecting backfill.** Subscribe first; never the other way around.
