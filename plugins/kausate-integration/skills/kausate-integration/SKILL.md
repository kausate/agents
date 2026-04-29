---
name: kausate-integration
description: Integrate the Kausate company-data API. Generate a typed client from the live OpenAPI spec, set up async + webhook handling, use customerId and customerReference correctly, and fall back to polling only when webhooks aren't possible.
---

# Kausate integration

Kausate is a real-time company data API for KYB and onboarding. It pulls live data from official business registries across multiple jurisdictions. Most data-retrieval endpoints are **async** because government registries are slow and unreliable — your integration must handle this correctly.

This skill installs the production-ready patterns. Follow it in order.

## 1. Authentication

All requests use an API key sent via the `X-API-Key` header:

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" https://api.kausate.com/v2/...
```

- Get an API key from [kausate.com/signup](https://www.kausate.com/signup) → dashboard → API keys.
- **Never commit keys.** Read from environment variables only (`KAUSATE_API_KEY` is the conventional name). Add `.env` to `.gitignore` if it isn't already.
- Rotate keys via the dashboard if leaked.

## 2. Generate a typed client from `openapi.json`

The Kausate API publishes an OpenAPI spec at `https://api.kausate.com/openapi.json`. The endpoint requires the `X-API-Key` header (it isn't anonymous).

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" \
  https://api.kausate.com/openapi.json \
  -o openapi.json
```

Then generate a typed client. Pick the matching ecosystem:

**TypeScript:**
```bash
npx openapi-typescript openapi.json -o src/lib/kausate.d.ts
```

For a runtime client (not just types), use `openapi-fetch`:
```bash
npm install openapi-fetch
```
```ts
import createClient from "openapi-fetch";
import type { paths } from "./lib/kausate";

export const kausate = createClient<paths>({
  baseUrl: "https://api.kausate.com",
  headers: { "X-API-Key": process.env.KAUSATE_API_KEY! },
});
```

**Python:**
```bash
uvx --from openapi-python-client openapi-python-client generate --path openapi.json
```

**Go:**
```bash
go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest \
  -package kausate -generate types,client openapi.json > kausate.gen.go
```

**Java / Kotlin:**
```bash
npx @openapitools/openapi-generator-cli generate \
  -i openapi.json -g java -o src/main/java/kausate
```

### Pin the API version

Kausate uses date-based API versioning (Cadwyn). Pin a version in your client by sending the `X-API-Version` header so request/response shapes don't drift under you:

```
X-API-Version: 2026-05-01
```

If you don't pin, you get the latest version and breaking changes can surface during a migration. **Always pin in production.** The available versions are listed under `info` in `openapi.json` and on the docs site.

## 3. Async + webhooks (the only production pattern)

Most endpoints — Company Report, Documents, UBO, Shareholder Graph — are async. They return immediately with an `orderId`; the result lands later via webhook (or by polling if you can't accept webhooks).

**Hard rule for production: webhooks first, polling only as a fallback, never sync.**

### Webhook receiver — production-ready checklist

Set up an HTTPS endpoint on your side that accepts `POST` requests. It must:

1. **Authenticate the request via custom headers.** Kausate does **not** sign payloads with HMAC. Instead, when you create a webhook subscription you supply custom headers (e.g. `Authorization: Bearer <secret>`) that Kausate stores encrypted and replays on each delivery. Verify those headers in your handler.
2. **Respond 2xx fast.** Acknowledge within seconds and process asynchronously on your side. Long-running work in the request handler will hit the 30-second per-attempt delivery timeout and trigger needless retries.
3. **Be idempotent on `orderId`.** Kausate retries up to **50 times** with exponential backoff (1 s → max 4 h interval, 2× multiplier). Same-day request deduplication can also produce duplicate deliveries. Track which `orderId`s you've already processed and short-circuit duplicates.
4. **Handle every terminal status**, not just `completed` (see §5).

### Subscribe a webhook

```bash
curl -X POST https://api.kausate.com/v2/webhooks \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Order completions",
    "url": "https://your-app.example.com/webhooks/kausate",
    "apiVersion": "2026-05-01",
    "customHeaders": {
      "Authorization": "Bearer <your-shared-secret>"
    }
  }'
```

A single subscription receives completions for **all** order types your org places (reports, documents, UBO, shareholder graph, financials). There is no event-type filter; switch on the payload's `result.type` or the `status` field instead.

### Place an async order

```bash
curl -X POST https://api.kausate.com/v2/companies/co_de_7KGHtucR88u2omSx3KhaoH/report \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "X-API-Version: 2026-05-01" \
  -H "X-Customer-Id: end-customer-789" \
  -H "Content-Type: application/json" \
  -d '{
    "customerReference": "kyc-case-12345"
  }'
```

Response (immediately):

```json
{
  "orderId": "ord_xyz789",
  "customerReference": "kyc-case-12345",
  "customerId": "end-customer-789",
  "status": "running"
}
```

### Webhook payload

When the order finishes, Kausate `POST`s the full `ExecutionResponse` to your webhook URL:

```json
{
  "orderId": "ord_xyz789",
  "kausateId": "co_de_7KGHtucR88u2omSx3KhaoH",
  "customerReference": "kyc-case-12345",
  "customerId": "end-customer-789",
  "status": "completed",
  "requestTime": "2026-04-27T10:30:00Z",
  "responseTime": "2026-04-27T10:30:08Z",
  "result": { "...": "product-specific" },
  "error": null,
  "currentActivity": null
}
```

Match webhook deliveries back to the originating request via `orderId` (always present) or `customerReference` (your value, also always echoed back).

## 4. `customerReference` vs `customerId` — use both, deliberately

Both are optional, both are URL-safe strings (`[a-zA-Z0-9_-]`), both capped at **150 characters**. They have distinct purposes:

| Field | What it is | Typical value | Set per | How to send |
|---|---|---|---|---|
| `customerReference` | Per-request correlation id you generate | KYC case id, order id from your system | request | JSON body field |
| `customerId` | Persistent identifier for **your end customer** | the user / tenant / account in your product | end customer | `X-Customer-Id` header |

Both round-trip: they appear in the synchronous `ProductOrderResponse`, in the typed run-status response, and in the webhook payload. Use them to:

- **`customerReference`** — correlate an async webhook delivery back to the originating request. Set this in the JSON body. Always set this in production.
- **`customerId`** — partition usage by your own customers (cost attribution, customer-scoped reporting, customer-scoped webhook routing on your side). Send via the `X-Customer-Id` header whenever the request is on behalf of a specific end customer.

`customerId` is sent via the `X-Customer-Id` HTTP header — it is **not** a JSON body field. This makes it straightforward for a thin gateway to inject the customer id without rewriting request bodies.

## 5. Handle all `status` values

Async runs finish in one of six states. Treat anything other than `completed` as not-success:

| Status | Meaning | Action |
|---|---|---|
| `running` | Still processing | Wait for webhook or keep polling |
| `completed` | Success — `result` is populated | Use it |
| `failed` | Permanent failure — `error` populated | Surface to the user; consider retry with `bypassCache: true` |
| `canceled` | Order canceled and acknowledged | Treat as not-found from your side |
| `terminated` | Forcefully terminated (rare) | Same as failed |
| `timedOut` | Reached its time limit | Retry — registries can be slow; the next attempt may succeed |

Don't write code that only checks `status === "completed"`. Make the non-completed branch explicit.

## 6. Polling — fallback only

Use polling **only when** you can't accept inbound webhooks (no public HTTPS endpoint, restricted networks, dev/test environments).

```bash
curl https://api.kausate.com/v2/companies/report/$ORDER_ID \
  -H "X-API-Key: $KAUSATE_API_KEY"
```

Polling endpoints follow the pattern `GET /v2/companies/{family}/{orderId}` — see `openapi.json` for the full list (e.g. `/v2/companies/documents/{orderId}`, `/v2/companies/shareholder-graph/{orderId}`).

**Recommended schedule** — exponential backoff with a hard cap:

- Start at **5 s** between polls.
- Multiply by **1.2** each attempt.
- Cap at **15 s**.
- Give up after **30 minutes** total (some workflows take longer; if you hit the cap and the order is still `running`, surface that to the user rather than waiting forever).

**Belt-and-braces:** if you've subscribed a webhook AND are running a periodic reconciliation job that polls any orders still `running` past their expected window, you have a self-healing system — webhook delivery failures don't strand orders forever.

## 7. Anti-patterns — never generate this code

- **Sync mode (`?sync=true`) in production.** Government registries are unreliable; sync requests will return HTTP 408 even when the data would eventually become available. Use sync only in tests and prototypes.
- **Polling faster than every 5 seconds.** It doesn't get you results faster (the registry call has its own latency) and burns rate budget.
- **Treating `completed` as the only outcome.** Every closed status (`failed`, `canceled`, `terminated`, `timedOut`) needs its own handling.
- **Hardcoding the API base URL across environments.** Use a config var (`KAUSATE_API_BASE_URL`) so dev/staging/prod can point at the right host.
- **Storing the API key in source.** Read from env. Never check `.env` files into git.
- **Skipping `X-API-Version`.** You will get bitten when the latest version changes.
- **Doing real work inside the webhook handler.** Acknowledge fast; queue.
- **Assuming a webhook won't be redelivered.** It will. Be idempotent on `orderId`.

## 8. Production checklist

Before you ship:

- [ ] API key in env vars, never in source.
- [ ] Pin `X-API-Version` to a specific date.
- [ ] Webhook receiver auths via the custom headers you set at subscription time.
- [ ] Webhook receiver returns 2xx in under 5 s and queues real work.
- [ ] Idempotent on `orderId`.
- [ ] All six `status` values handled distinctly.
- [ ] Polling is a fallback path only; webhook is the primary signal.
- [ ] Reconciliation job for orders still `running` past their expected window.
- [ ] `customerReference` set per request, `customerId` set per end customer.
- [ ] Retry on 5xx with backoff; do not retry on 4xx.
- [ ] Log `orderId` on every request and every webhook delivery for tracing.

## 9. Reference

- **Live OpenAPI spec** (auth required): `https://api.kausate.com/openapi.json`
- **Human-readable docs**: <https://docs.kausate.com>
- **Async & sync processing**: <https://docs.kausate.com/api-reference/async-sync>
- **Webhooks reference**: <https://docs.kausate.com/api-reference/webhooks>
- **Getting started walkthrough**: <https://docs.kausate.com/getting-started>
- **Changelog**: <https://docs.kausate.com/changelog>
- **Doc index for AI tools**: <https://docs.kausate.com/llms.txt>

If anything in this skill conflicts with what `openapi.json` says, the OpenAPI spec wins — it's generated from the live code.
