# Async lifecycle, webhooks, and polling fallback

Most data endpoints are async because government registries time out, rate-limit, and have weekend/maintenance outages. The async flow:

1. Your code `POST`s to a data endpoint with `kausateId` in the body. The API returns `{orderId, status: "running"}` immediately.
2. Kausate's worker fetches data from the registry in the background (this can take seconds to minutes; shareholder-graph can take longer).
3. When the order finishes, Kausate `POST`s the full result to your registered webhook URL. If you don't have a webhook, you poll.

**Hard rule for production: webhooks first. Polling is a fallback only. Sync mode is for prototypes.**

## Subscribe a webhook

```bash
curl -X POST https://api.kausate.com/v2/webhooks \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Order completions",
    "url": "https://your-app.example.com/webhooks/kausate",
    "customHeaders": {
      "Authorization": "Bearer <your-shared-secret>"
    }
  }'
```

Notes:

- A single subscription receives completions for **all** order types your org places. There is no event-type filter. Switch on `result.type` in your handler:

| `result.type` | Comes from |
|---|---|
| `liveSearch` | `POST /v2/companies/search` |
| `companyReport` | `POST /v2/companies/report` |
| `financials` | `POST /v2/companies/finance` |
| `uboReport` | `POST /v2/companies/ubo` |
| `shareholderGraph` | `POST /v2/companies/shareholder-graph` |
| `listDocuments` | `POST /v2/companies/documents/list` |
| `document` | `POST /v2/companies/documents` |
| `person` | `GET /v2/companies/search/person` (rare in async path) |

A typical receiver pattern:

```ts
switch (payload.result?.type) {
  case "companyReport":   handleReport(payload); break;
  case "uboReport":       handleUbo(payload); break;
  case "shareholderGraph":handleGraph(payload); break;
  case "listDocuments":   handleDocList(payload); break;
  case "document":        handleDocument(payload); break;
  // ...
  default:
    // null result = error path; check payload.status / payload.error
}
```

If `payload.result` is `null`, the order didn't reach a successful state — branch on `payload.status` (see `status-and-dedup.md`) and use `payload.error` for the user-visible message.

- All `ACTIVE` webhooks for the org receive every event. Multi-tenant routing happens in your handler — typically by inspecting `customerReference` or `customerId`.
- The `apiVersion` field on the subscription body is **deprecated**. Webhook payload shape is determined by the `Kausate-Version` header that was on the *original order request*, falling back to your org's default. Pin per-request, not per-subscription.
- `customHeaders` are stored encrypted server-side and replayed on every delivery. Six headers are protected and cannot be overridden: `content-type`, `host`, `content-length`, `transfer-encoding`, `connection`, `kausate-version`.

## Manage webhook subscriptions

The subscription returns an `id` (UUID), `status` (`active` | `inactive`), and `url`. Persist the id — you'll need it to update or delete the subscription.

```bash
# List
curl -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  https://api.kausate.com/v2/webhooks

# Get one
curl -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  https://api.kausate.com/v2/webhooks/$WEBHOOK_ID

# Pause without deleting (status: inactive — no deliveries until you re-activate)
curl -X PUT -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"status":"inactive"}' \
  https://api.kausate.com/v2/webhooks/$WEBHOOK_ID

# Delete (returns 204)
curl -X DELETE -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  https://api.kausate.com/v2/webhooks/$WEBHOOK_ID
```

Pausing is safer than deleting if you're rotating receivers — pause the old one, register the new one, verify it works, then delete the old one. Orders placed during a fully-paused window will not redeliver when you reactivate (no backfill — see below).

## Place an async order

Path style is `2026-05-01`: `kausateId` in the body, not the URL.

```bash
curl -X POST https://api.kausate.com/v2/companies/report \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "X-Customer-Id: end-customer-789" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId": "co_de_7KGHtucR88u2omSx3KhaoH",
    "customerReference": "kyc-case-12345"
  }'
```

Immediate response:

```json
{
  "orderId": "ord_xyz789",
  "customerReference": "kyc-case-12345",
  "customerId": "end-customer-789",
  "status": "running"
}
```

## Webhook receiver — production-ready checklist

Set up an HTTPS endpoint on your side that accepts `POST` requests. It must:

1. **Authenticate via the custom headers you set at subscription time.** Kausate does **not** HMAC-sign payloads. Verify the `Authorization` (or whatever) header you registered.
2. **Acknowledge with 2xx fast.** The HTTP request timeout is **30 s**. Long-running work will trip retries needlessly. Queue real work and respond immediately.
3. **Be idempotent on `orderId`.** Kausate retries up to **50 times** with exponential backoff (1 s initial, 2× multiplier, 4 h max interval). Same-day dedup can also produce duplicate deliveries.
4. **Handle every closed status, not just `completed`.** See `status-and-dedup.md`.

## Headers Kausate adds to outgoing webhook requests

Kausate sends only two headers beyond your custom ones:

- `Content-Type: application/json`
- `Kausate-Version: <delivered_version>` — the API version this payload was migrated to (matches the order's request version)

There is no Kausate-supplied delivery-id, request-id, or signature header. Your auth + idempotency must be handled via your custom headers + the payload's `orderId`.

## Webhook payload (`ExecutionResponse`)

```json
{
  "orderId": "ord_xyz789",
  "kausateId": "co_de_7KGHtucR88u2omSx3KhaoH",
  "customerReference": "kyc-case-12345",
  "customerId": "end-customer-789",
  "status": "completed",
  "requestTime": "2026-04-29T10:30:00Z",
  "responseTime": "2026-04-29T10:30:08Z",
  "result": { "type": "companyReport", "...": "product-specific" },
  "error": null,
  "currentActivity": null
}
```

Match deliveries back to your originating request via `orderId` (always present) or `customerReference` (your value, always echoed back when supplied).

## No backfill — register webhooks before placing orders

If you place an order, then add a webhook subscription, the result is **not** delivered retroactively. The notification fires once at workflow completion. Subscribe webhooks during onboarding, not after orders go out. For orders that fired before a subscription existed, poll with `GET /v2/companies/{family}/{orderId}`.

## Polling — fallback only

Use polling only when you can't accept inbound webhooks (no public HTTPS endpoint, restricted networks, dev / test).

> **Don't register a webhook URL the receiver can't actually serve.** Kausate retries undelivered webhooks up to **50 times** with exponential backoff (1 s initial → 4 h max). A URL that's unreachable (closed firewall, sleeping tunnel, lambda cold-start failure, expired TLS cert) is actively harmful — it burns the retry budget for every order without delivering anything. If the receiver can't reach the public internet, **don't subscribe a webhook**; go polling-only at `GET /v2/companies/{family}/{orderId}`. You can subscribe a real receiver later — but until then, no subscription is better than a dead one.

```bash
curl https://api.kausate.com/v2/companies/report/$ORDER_ID \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"
```

Polling endpoints follow `GET /v2/companies/{family}/{orderId}` — `report`, `documents`, `documents/list`, `finance`, `ubo`, `shareholder-graph`, `search`. The response shape is the same `ExecutionResponse` as the webhook payload.

**Recommended schedule** — exponential backoff with a hard cap:

- Start at **5 s** between polls.
- Multiply by **1.2** each attempt.
- Cap at **15 s**.
- Give up after **30 minutes** total. Async workflows have a 7-day execution timeout server-side, but if you hit your local cap and the order is still `running`, surface that to the user and let them retry asynchronously rather than blocking forever.

**Belt-and-braces:** when possible run BOTH a webhook AND a periodic reconciliation job that polls any orders still `running` past their expected window. That way an occasional missed webhook delivery doesn't strand orders.
