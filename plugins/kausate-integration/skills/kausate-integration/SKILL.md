---
name: kausate-integration
description: Integrate the Kausate company-data API (KYB / business registries / shareholder graphs / UBO / documents / change monitoring). Generate a typed client from the live OpenAPI spec, set up async + webhook handling, use customerId and customerReference correctly, fall back to polling only when webhooks aren't possible. Use this skill whenever the user asks to integrate Kausate, fetch company data, do KYB, look up companies in business registries, retrieve shareholder graphs / UBO / company reports / annual accounts, monitor a company for changes (insolvency, dissolution, director changes, ownership changes), wire up Kausate webhooks, or react to business-registry events.
---

# Kausate integration

Kausate is a real-time KYB / company-data API. It pulls live data from official business registries across 50+ jurisdictions (DE Handelsregister, GB Companies House, FR INPI, NL KVK, etc.). Most data-retrieval endpoints are **async** because government registries are slow and unreliable — your integration must handle this correctly to be production-ready.

This skill encodes the production-grade patterns. Follow it in order. **If anything here disagrees with `https://api.kausate.com/openapi.json`, the OpenAPI spec wins** — it's generated from the live code.

## 1. Authentication

All requests use an API key sent via the `X-API-Key` header:

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" https://api.kausate.com/v2/...
```

- Get an API key from [kausate.com/signup](https://www.kausate.com/signup) → dashboard → API keys.
- **Read from environment variables only.** Never commit keys. Add `.env*` to `.gitignore`.
- Rotate via the dashboard if a key leaks.
- API keys are org-scoped. Auth is `X-API-Key` only — no OAuth, no JWT for the public API.
- **No sandbox or test environment.** Every call hits production registries and consumes credits. Use small test budgets during development.

## 2. Generate a typed client from `openapi.json`

Kausate publishes its OpenAPI spec at `https://api.kausate.com/openapi.json`. The endpoint requires the `X-API-Key` header (it isn't anonymous):

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" \
  https://api.kausate.com/openapi.json \
  -o openapi.json
```

Then generate a typed client. Pick the matching ecosystem:

**TypeScript** — types only:
```bash
npx openapi-typescript openapi.json -o src/lib/kausate.d.ts
```

**TypeScript** — runtime client (recommended):
```bash
npm install openapi-fetch
```
```ts
import createClient from "openapi-fetch";
import type { paths } from "./lib/kausate";

export const kausate = createClient<paths>({
  baseUrl: "https://api.kausate.com",
  headers: {
    "X-API-Key": process.env.KAUSATE_API_KEY!,
    "Kausate-Version": "2026-05-01",
  },
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

### Pin the API version with `Kausate-Version`

Kausate uses date-based API versioning (Cadwyn). Pin a version on every request via the `Kausate-Version` header so request/response shapes don't drift under you:

```
Kausate-Version: 2026-05-01
```

If you don't pin, you get the org's default version (or the latest if no default is set), and breaking changes can land during a major-release window. **Always pin in production.** Read the `info` block of `openapi.json` for the version list.

> **Important:** Different API versions expose different paths. In `2025-04-01` the report endpoint was `POST /v2/companies/{kausateId}/report`; in `2026-05-01` it's `POST /v2/companies/report` with `kausateId` in the body. Always match your path style to the version you've pinned. Generating the client from `openapi.json` while sending the matching `Kausate-Version` header keeps these in sync automatically.

## 3. Endpoint inventory (latest version `2026-05-01`)

The full source of truth is `openapi.json`; this is the map an integrator usually needs.

### Search & discovery

| Endpoint | Purpose | Mode | Latency |
|---|---|---|---|
| `GET /v2/companies/search/autocomplete` | Type-ahead during form input | sync | < 100 ms |
| `POST /v2/companies/search` | Live registry search by name / number / advanced query | **async by default**, sync available at `/search/sync` | 2–10 s live |
| `GET /v2/companies/search/person` | Find companies by legal-rep name + DOB | sync | 1–5 s |
| `POST /v2/companies/prefill` | Form-prefill from a known `kausateId`. Index hit + live fallback. Best onboarding UX. | always sync | < 200 ms typical |

Search request body fields: `companyName`, `companyNumber`, `advancedQuery` (jurisdiction-specific structured query), `jurisdictionCode`, `customerReference`. Use `advancedQuery` when you have native registry numbers — it avoids fuzzy matching.

### Translating a native registry ID to a `kausateId`

`prefill` and every data endpoint require a `kausateId` in the body. They do **not** accept `companyNumber` or `jurisdictionCode` — submitting those fields fails 422 (`extra_forbidden`). To go from a native ID (KVK number, CRN, SIREN, HRB, etc.) to a `kausateId`, use `search`:

```bash
# Step 1 — translate native id → kausateId via search
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyNumber":"33255959","jurisdictionCode":"nl"}'
# → result.searchResults[0].kausateId

# Step 2 — use that kausateId everywhere else
curl -X POST https://api.kausate.com/v2/companies/prefill \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"kausateId":"co_nl_..."}'
```

Cache the `(native_id, kausateId)` pair on your side. The mapping is stable, and re-running search on every call wastes credits.

### Company data (async by default, sync available)

| Endpoint | Purpose |
|---|---|
| `POST /v2/companies/report` | Full company report from the registry (legal name, addresses, identifiers, capital, legal reps, shareholders) |
| `POST /v2/companies/finance` | Financials / annual accounts when available |
| `POST /v2/companies/ubo` | Ultimate Beneficial Owners |
| `POST /v2/companies/shareholder-graph` | Multi-level ownership graph. Always async. Configurable `maxDepth` (1–7), `enriched` (governance enrichment), `retrievalTimeout` (minutes) |
| `POST /v2/companies/documents/list` | List the documents available for a company |
| `POST /v2/companies/documents` | Retrieve a specific document (binary served via pre-signed URL in the result) |

All of the above accept the same standard body fields:

- `kausateId` (required) — Kausate's company identifier (`co_{jurisdiction}_{id}`)
- `customerReference` (optional) — your per-request correlation id (see §6)
- `bypassCache` (optional, where supported) — skip same-day dedup, force a fresh registry call
- Plus the workflow-specific fields above (e.g. `maxDepth` on shareholder-graph)

For each async endpoint there's a matching sync variant at `/{family}/sync` and a polling endpoint at `GET /v2/companies/{family}/{orderId}` (see §5).

### Webhooks, monitors, analytics, jurisdictions

| Endpoint | Purpose |
|---|---|
| `POST /v2/webhooks` / `GET` / `PUT /{id}` / `DELETE /{id}` | Manage webhook subscriptions |
| `POST /v2/monitors` / `GET` / `GET /{id}` / `DELETE /{id}` | Monitor a company on a cron schedule, fire webhook on detected changes |
| `GET /v2/analytics/summary` | Usage / cost rollup, filterable by date range, workflowType, sku, customerId |
| `GET /v2/analytics/timeseries` | Same metrics over time |
| `GET /v2/analytics/breakdowns` | Pivot by jurisdiction, sku, customerId |
| `GET /v2/platform/jurisdictions` | Capability matrix per jurisdiction (which products are supported, what document types exist) |

## 4. The async + webhook pattern (the only production setup)

> **This section is about order-completion webhooks** (`POST /v2/webhooks` subscriptions, `ExecutionResponse` payload, camelCase). **Monitor change-event webhooks are a separate channel** with a different payload shape — see §10. Don't reuse one handler for both.

Most data endpoints are async because government registries time out, rate-limit, and have weekend/maintenance outages. The async flow:

1. Your code `POST`s to a data endpoint with `kausateId` in the body. The API returns `{orderId, status: "running"}` immediately.
2. Kausate's worker fetches data from the registry in the background (this can take seconds to minutes; shareholder-graph can take longer).
3. When the order finishes, Kausate `POST`s the full result to your registered webhook URL. If you don't have a webhook, you poll (§5).

**Hard rule for production: webhooks first. Polling is a fallback only. Sync mode is for prototypes.**

### Subscribe a webhook

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

If `payload.result` is `null`, the order didn't reach a successful state — branch on `payload.status` (see §7) and use `payload.error` for the user-visible message.
- All `ACTIVE` webhooks for the org receive every event. Multi-tenant routing happens in your handler — typically by inspecting `customerReference` or `customerId`.
- The `apiVersion` field on the subscription body is **deprecated**. Webhook payload shape is determined by the `Kausate-Version` header that was on the *original order request*, falling back to your org's default. Pin per-request, not per-subscription.
- `customHeaders` are stored encrypted server-side and replayed on every delivery. Six headers are protected and cannot be overridden: `content-type`, `host`, `content-length`, `transfer-encoding`, `connection`, `kausate-version`.

### Manage webhook subscriptions

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

Pausing is safer than deleting if you're rotating receivers — pause the old one, register the new one, verify it works, then delete the old one. Orders placed during a fully-paused window will not redeliver when you reactivate (no backfill — see §4).

### Place an async order

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

### Webhook receiver — production-ready checklist

Set up an HTTPS endpoint on your side that accepts `POST` requests. It must:

1. **Authenticate via the custom headers you set at subscription time.** Kausate does **not** HMAC-sign payloads. Verify the `Authorization` (or whatever) header you registered.
2. **Acknowledge with 2xx fast.** The HTTP request timeout is **30 s**. Long-running work will trip retries needlessly. Queue real work and respond immediately.
3. **Be idempotent on `orderId`.** Kausate retries up to **50 times** with exponential backoff (1 s initial, 2× multiplier, 4 h max interval). Same-day dedup can also produce duplicate deliveries.
4. **Handle every closed status, not just `completed`.** See §7.

### Headers Kausate adds to outgoing webhook requests

Kausate sends only two headers beyond your custom ones:

- `Content-Type: application/json`
- `Kausate-Version: <delivered_version>` — the API version this payload was migrated to (matches the order's request version)

There is no Kausate-supplied delivery-id, request-id, or signature header. Your auth + idempotency must be handled via your custom headers + the payload's `orderId`.

### Webhook payload (`ExecutionResponse`)

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

### No backfill — register webhooks before placing orders

If you place an order, then add a webhook subscription, the result is **not** delivered retroactively. The notification fires once at workflow completion. Subscribe webhooks during onboarding, not after orders go out. For orders that fired before a subscription existed, poll with `GET /v2/companies/{family}/{orderId}`.

## 5. Polling — fallback only

Use polling only when you can't accept inbound webhooks (no public HTTPS endpoint, restricted networks, dev / test).

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

## 6. `customerReference` vs `customerId` — use both, deliberately

Both are optional, both are URL-safe strings (`[a-zA-Z0-9_-]`), both capped at **150 characters**. They have distinct purposes and they're sent at different layers.

| Field | What it is | Typical value | Set per | How to send |
|---|---|---|---|---|
| `customerReference` | Per-request correlation id you generate | KYC case id, order id from your system | request | JSON body field |
| `customerId` | Persistent identifier for **your end customer** | the user / tenant / account in your product | end customer | `X-Customer-Id` header |

`customerId` is a header, not a body field. Sending it in the body fails Pydantic validation because the request models are `extra=forbid`. The header makes it easy for a thin gateway to inject a per-end-customer id without rewriting bodies.

Both round-trip: they appear in the immediate `ProductOrderResponse`, in the polling response, and in the webhook payload. Use them like this:

- **`customerReference`** — set per request. Lets you correlate a webhook delivery back to the originating call. Always set this in production.
- **`customerId`** — set per end customer. Lets you partition usage and cost in `/v2/analytics/breakdowns?groupBy=customerId`, and is the primary key your handler uses to route results to the right tenant / user.

## 7. Order status values — handle all six

Async runs finish in one of six states. Treat anything other than `completed` as not-success.

| Status | Meaning | Action |
|---|---|---|
| `running` | Still processing | Wait for webhook or keep polling |
| `completed` | Success — `result` is populated | Use it |
| `failed` | Permanent failure — `error` populated | Surface to the user; retry with `bypassCache: true` if appropriate (registry was down, etc.) |
| `canceled` | Order canceled and acknowledged | Treat as not-found from your side |
| `terminated` | Forcefully terminated (rare) | Same as `failed` |
| `timedOut` | Reached its time limit (workflow-level 7 days; can hit registry-side limits earlier) | Retry — registries can be slow; the next attempt may succeed |

Don't write code that only checks `status === "completed"`. Make the non-completed branch explicit.

## 8. Same-day dedup and `bypassCache`

If you place the same request twice on the same UTC calendar day (same `kausateId`, same workflow, same `customerReference`, same `customerId`), Kausate returns the existing `orderId` instead of starting a new workflow. This protects against duplicate billing on retries.

If you genuinely need a fresh registry call (e.g. data was missing, registry was down on the first try), set `bypassCache: true` on the request body. This skips the dedup key, starts a new workflow, and consumes new credits.

`bypassCache` does **not** bypass any internal result cache — only the same-day dedup. The result you get back is always live from the registry.

## 9. Documents — the two-step workflow

Document retrieval is two async orders, in this order:

1. **List documents available for a company:**
   ```bash
   curl -X POST https://api.kausate.com/v2/companies/documents/list \
     -H "X-API-Key: $KAUSATE_API_KEY" \
     -H "Kausate-Version: 2026-05-01" \
     -d '{"kausateId":"co_de_...","customerReference":"docs-list-1"}'
   ```
   On completion, `result` has shape (type `listDocuments`):
   ```json
   {
     "type": "listDocuments",
     "kausateId": "co_de_...",
     "documents": [
       {
         "kausateDocumentId": "doc_...",
         "title": "Aktueller Abdruck",
         "documentType": "currentExtract",
         "publicationType": null,
         "publicationDate": "2024-03-15",
         "fileType": "pdf",
         "source": "de-handelsregister"
       }
     ],
     "totalCount": 1,
     "indexedAt": "2024-...",
     "expiresAt": "2024-..."
   }
   ```
   `documentType` values include `annualAccounts`, `currentExtract`, `articlesOfAssociation`, `shareholderList`, `chronologicalExtract`, `officialFilings`, `registrationDetails`, `beneficialOwnersDetails`, `annualReturn`. `documentType`/`publicationDate` are sometimes `null`; filter accordingly.

2. **Retrieve a specific document by `kausateDocumentId`:**
   ```bash
   curl -X POST https://api.kausate.com/v2/companies/documents \
     -H "X-API-Key: $KAUSATE_API_KEY" \
     -H "Kausate-Version: 2026-05-01" \
     -d '{
       "kausateId":"co_de_...",
       "kausateDocumentId":"<from-list-response>",
       "customerReference":"docs-fetch-1"
     }'
   ```
   On completion, `result` has shape (type `document`):
   ```json
   {
     "type": "document",
     "kausateDocumentId": "doc_...",
     "documentType": "currentExtract",
     "downloadLink": "https://kausate-...s3.../...?X-Amz-Signature=...",
     "expiresAt": "2024-...",
     "contentType": "application/pdf",
     "fileName": "current-extract.pdf"
   }
   ```
   `downloadLink` is a pre-signed S3-style URL with an `expiresAt` timestamp. **Webhook payloads carry the URL, not the bytes.** Fetch the bytes from `downloadLink` directly (no `X-API-Key` needed on that fetch — the URL is signed) before `expiresAt`.

## 10. Monitors — change detection on a schedule

A **monitor** watches one company across one or more **sources** and emits **events** when it detects a change. Each event is classified into a generic, jurisdiction-agnostic taxonomy (`event_code` + `category`) so your routing logic doesn't need a per-jurisdiction mapping table. Per-event billing: 1 credit (`AAKMON`) per scheduled tick of a per-company source, plus 1 credit (`AAKEVT`) per delivered webhook event (refunded if every retry fails).

> **Two distinct webhook channels.** The org-wide `POST /v2/webhooks` subscription (§4) delivers **async order completions** (`ExecutionResponse` payloads — camelCase, `result.type` discriminator). Monitor change events are delivered to the **per-monitor `webhookUrl`** you set on `POST /v2/monitors` (`monitor.change_detected` payload — **snake_case**, completely different shape). Don't reuse one handler for both — switch on the path your endpoint receives or use separate URLs.

### Source modes

| Mode | When the source ticks | Examples |
|---|---|---|
| `per_company_pull` | Driven by your monitor's `scheduleCron`. Calls a registry capability and diffs the result against the last observation. | `company_report`, `shareholder_graph` |
| `global_feed` | Runs centrally whenever the upstream publishes; cron is **ignored**. Each item is matched to companies in our index. | `de_insolvenzbekanntmachungen.feed` (German §9 InsO portal) |

### Discover the sources that apply to a company

Always run this before creating a monitor — sources are gated by jurisdiction and the company's integration capabilities. Posting an inapplicable source returns 422 with a precise error.

```bash
curl -G https://api.kausate.com/v2/monitors/sources \
  --data-urlencode "kausateId=co_de_2tT6getO1qTAvO3iaGnju6" \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"
```

Returns only the sources valid for that company:

```json
{
  "sources": [
    { "name": "company_report", "mode": "per_company_pull",
      "producesCategories": ["status","address","legalRepresentatives","financial","other","disappeared"] },
    { "name": "shareholder_graph", "mode": "per_company_pull",
      "producesCategories": ["ownership"] },
    { "name": "de_insolvenzbekanntmachungen.feed", "mode": "global_feed",
      "producesCategories": ["status","financial","other"] }
  ]
}
```

For the global catalog (not filtered by company), use `GET /v2/events/sources`.

### Categories and event codes

Every event is one of seven `category` values, each bundling a handful of specific `event_code`s:

| Category | Examples of event codes |
|---|---|
| `status` | `INSOLVENCY_OPENED`, `INSOLVENCY_DECISION`, `STATUS_CHANGED` |
| `address` | `REGISTERED_ADDRESS_CHANGED`, `BUSINESS_ADDRESS_CHANGED` |
| `ownership` | `SHAREHOLDER_ADDED`, `UBO_PERCENTAGE_CHANGED` |
| `financial` | `SHARE_CAPITAL_CHANGED`, `FINANCIAL_FILING_PUBLISHED` |
| `legalRepresentatives` | `DIRECTOR_ADDED`, `AUTHORIZED_SIGNATORY_CHANGED` |
| `other` | `NAME_CHANGED`, `LEGAL_FORM_CHANGED` |
| `disappeared` | `COMPANY_DISSOLVED`, `COMPANY_STRUCK_OFF`, `COMPANY_LIQUIDATED` |

Don't hardcode this list — fetch the current taxonomy at startup and build your routing table from it:

```bash
curl https://api.kausate.com/v2/events/taxonomy \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"
```

```json
{
  "categories": ["status","address","ownership","financial","legalRepresentatives","other","disappeared"],
  "eventCodes": [
    { "eventCode": "INSOLVENCY_OPENED", "category": "status" },
    { "eventCode": "COMPANY_DISSOLVED", "category": "disappeared" },
    "..."
  ]
}
```

### Create a monitor

```bash
curl -X POST https://api.kausate.com/v2/monitors \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId": "co_de_2tT6getO1qTAvO3iaGnju6",
    "sources": [
      "company_report",
      "shareholder_graph",
      "de_insolvenzbekanntmachungen.feed"
    ],
    "scheduleCron": "0 9 * * *",
    "webhookUrl": "https://your-app.example.com/webhooks/kausate-monitors",
    "categoriesFilter": ["status", "ownership", "disappeared"],
    "autoDeactivateCategories": ["disappeared"]
  }'
```

Body fields:

- `kausateId` (required) — Kausate company id.
- `sources` (required, ≥1) — names from the per-company source list above. Mixing `per_company_pull` and `global_feed` sources in one monitor is fine.
- `scheduleCron` (required) — standard 5-field cron, UTC, validated server-side. **Drives `per_company_pull` sources only**; ignored for `global_feed`. Required even when all sources are feeds (use any valid expression like `0 0 * * *`).
- `webhookUrl` (optional) — per-monitor delivery target for change events. Without it the monitor still records events (queryable via `GET /v2/monitors/{id}/events`) but nothing is pushed to you.
- `categoriesFilter` (optional, default = all) — only deliver events whose category is in this list.
- `eventCodesFilter` (optional, default = all) — only deliver events whose code is in this list. Both filters apply if both are set.
- `autoDeactivateCategories` (optional, default = `["disappeared"]`) — categories whose events flip the monitor to `is_active=false`. **Pass `[]` to opt out** — the empty list is the documented opt-out, distinct from omitting the field (which uses the default).

Filters apply at the **delivery boundary** — non-matching events are still persisted (visible via the events list and replayable), but no webhook is fired for them.

Response (`MonitorResponse` — fields actually returned):

```json
{
  "monitorId": "5d4241a1-...",
  "kausateId": "co_de_...",
  "companyName": "Bundesanzeiger Verlag GmbH",
  "sources": ["company_report","shareholder_graph","de_insolvenzbekanntmachungen.feed"],
  "categoriesFilter": ["status","ownership","disappeared"],
  "eventCodesFilter": null,
  "autoDeactivateCategories": ["disappeared"],
  "scheduleCron": "0 9 * * *",
  "webhookUrl": "https://your-app.example.com/webhooks/kausate-monitors",
  "isActive": true,
  "deactivationReason": null,
  "createdAt": "2026-05-04T22:00:00Z",
  "apiUrl": "https://api.kausate.com/v2/monitors/5d4241a1-..."
}
```

There is no `checkCount` / `lastCheckedAt` on v2 — observations are persisted per-source and surfaced via the events list.

### Webhook payload (`monitor.change_detected`)

Posted to the monitor's `webhookUrl`. Headers: `Content-Type: application/json`, `Kausate-Version: <version>`. **No HMAC; auth via custom headers is not supported on monitor webhooks today** — host the receiver on an unguessable HTTPS path, or front it with a secret-bearing reverse-proxy if you need stronger auth.

```json
{
  "event": "monitor.change_detected",
  "monitor_id": "5d4241a1-...",
  "kausate_id": "co_de_19IubOgcrWVQ7pRpNIBaDX",
  "event_id": "b2fa69b8-...",
  "event_code": "INSOLVENCY_DECISION",
  "category": "status",
  "severity": "info",
  "detected_at": "2026-05-04T22:00:00Z",
  "diff_path": null,
  "before": null,
  "after": {
    "aktenzeichen": "9 IN 1137/24",
    "publicationDate": "2026-05-02",
    "court": "Heilbronn",
    "publicationType": "Entscheidungen im Verfahren"
  },
  "metadata": {
    "source": "de_insolvenzbekanntmachungen.feed",
    "external_id": "9 IN 1137/24|2026-05-02|Entscheidungen im Verfahren",
    "jurisdiction": "de",
    "api_url": "https://api.kausate.com/v2/monitors/5d4241a1-.../events/b2fa69b8-..."
  }
}
```

Field shape per source mode:

- **Per-company-pull events** — `before`/`after` carry the changed values; `diff_path` points at the location in the canonical CompanyReport (e.g. `companyReport.basicInformation.companyStatus`).
- **Feed events** — `before` is `null` (the publication itself is the change); `after` carries the full publication payload; `metadata.source` and `metadata.external_id` identify the upstream item.

`event_code` and `category` are the **public contract** — route on these. `metadata.native_publication_type` (when present) carries the original upstream label for human debugging only — it's not a stable contract and changes with the upstream.

### Receiver requirements

- **Idempotent on `event_id`.** Delivery retries up to **50 times** with exponential backoff (1 s → 4 h max). Same event may arrive twice.
- **Acknowledge in 2xx fast** (same 30 s server-side timeout as order webhooks).
- **Branch on `event_code`** for specific transitions; **branch on `category`** for fan-out:

  ```ts
  // category-level fan-out
  switch (payload.category) {
    case "disappeared":          opsAlert(payload); break;
    case "ownership":            complianceReview(payload); break;
    case "status":
      if (payload.event_code.startsWith("INSOLVENCY_")) riskTeam(payload);
      break;
  }
  ```

### List + replay events

Every event is persisted and queryable, including those filtered out of webhook delivery. Useful for backfilling after an outage past the retry budget.

```bash
# Paginated list, with optional filters
curl "https://api.kausate.com/v2/monitors/$MONITOR_ID/events?limit=100&category=status&since=2026-05-01T00:00:00Z" \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"

# Re-trigger delivery for a single event (returns 202 {"status":"queued"})
curl -X POST "https://api.kausate.com/v2/monitors/$MONITOR_ID/events/$EVENT_ID/replay" \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"
```

Replay re-uses the existing `event_id`, so an idempotent receiver dedups it correctly.

### Auto-deactivation

By default, any event in the `disappeared` category flips the monitor's `is_active` to `false` and writes a `deactivationReason`. The intent is to stop ticking (and stop billing `AAKMON`) for a company that no longer exists. The monitor row is preserved — you can still list its events and replay deliveries — it just doesn't run new ticks.

To opt out entirely, send `"autoDeactivateCategories": []` at create time. To extend (e.g. also stop on insolvency), send the categories you want, e.g. `["disappeared", "status"]`.

### Billing

| SKU | When charged | Refund? |
|---|---|---|
| `AAKMON` | Each tick of a per-company source (5 credits per scheduled run) | No — the registry call already happened |
| `AAKEVT` | Each delivered event (1 credit) | Yes — refunded if every webhook retry fails |

Track usage like any other SKU via `/v2/analytics/breakdowns?groupBy=sku`.

### Common errors

| Status | Reason | Fix |
|---|---|---|
| `404 Company not found` | The `kausateId` doesn't exist in our index | Translate from a native id via `/v2/companies/search` first |
| `422 Unknown monitoring source` | Typo or fabricated source name | Discover via `/v2/events/sources` or the per-company variant |
| `422 Source not applicable to company` | Source is for a different jurisdiction or the company's integration doesn't expose the required capability | List valid sources via `/v2/monitors/sources?kausateId=…` |
| `422 Invalid cron expression` | Cron syntax | Use a valid 5-field cron (e.g. `0 9 * * *`) |
| `422 eventCodesFilter value not in enum` | Stale or misspelled event code | Fetch current codes from `/v2/events/taxonomy` |

## 11. Identifier formats per jurisdiction

Kausate normalizes companies to `kausateId = co_{jurisdiction}_{opaque}`. When *searching*, you can pass native registry identifiers via `companyNumber` or `advancedQuery`. The major formats:

| Jurisdiction | Native identifier | Format / example |
|---|---|---|
| `de` | Court-coded register number | `R2201_HRB 31248` (Cologne court HRB) — court code + register type + number |
| `gb` | Companies House CRN | 8 chars: `00445790` or `MC123456` |
| `fr` | SIREN / SIRET | 9-digit (SIREN) parent, 14-digit (SIRET) establishment |
| `nl` | KVK number | 8-digit |
| `at` | Firmenbuch FNR | digits + lowercase letter (`123456a`) |
| `it` | REA | regional code + number (`MI-1234567`); also Codice Fiscale (16) and P.IVA (11) |
| `ch` | UID | `CHE` + 9 digits |
| `be` | Enterprise number | 10 digits, optional dots: `1234.567.890` |
| `pl` | KRS / NIP / REGON | 10-digit (KRS) / 10-digit (NIP) / 9 or 14-digit (REGON) |
| `es` | NIF / CIF | letter + 7 digits + check digit |
| `se` | Organisationsnummer | 10 or 12 digits |
| `no` | Organisasjonsnummer | 9 digits |
| `fi` | Y-tunnus | 7 digits + `-` + check digit |

For everything else and for full coverage, call `GET /v2/platform/jurisdictions` to discover supported jurisdictions and their capabilities at runtime.

## 12. Error handling

Status codes returned by the public API:

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

## 13. Anti-patterns — never generate this code

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

## 14. Production checklist

Before you ship:

- [ ] API key in env vars, never in source.
- [ ] `Kausate-Version` pinned to a specific date on every request.
- [ ] Webhook subscribed *before* any orders go out.
- [ ] Webhook receiver authenticates via the custom headers you set at subscription time.
- [ ] Webhook receiver returns 2xx in < 5 s and queues real work.
- [ ] Idempotent on `orderId`.
- [ ] All six `status` values handled distinctly.
- [ ] Polling is a fallback path only; webhook is the primary signal.
- [ ] Reconciliation job for orders still `running` past their expected window.
- [ ] `customerReference` set per request, `customerId` set per end customer (header).
- [ ] Retry on 5xx with exponential backoff; do not retry on 4xx.
- [ ] Log `orderId` on every request and every webhook delivery for tracing.
- [ ] Cost monitoring via `/v2/analytics/summary` or the dashboard — every call costs credits.

## 15. Reference

- **Live OpenAPI spec** (auth required): `https://api.kausate.com/openapi.json`
- **Human-readable docs**: <https://docs.kausate.com>
- **Async & sync processing**: <https://docs.kausate.com/api-reference/async-sync>
- **Webhooks reference**: <https://docs.kausate.com/api-reference/webhooks>
- **Getting started walkthrough**: <https://docs.kausate.com/getting-started>
- **Jurisdiction guides**: <https://docs.kausate.com/guides/jurisdictions>
- **Per-page markdown twins** (for AI tools): append `.md` to any docs URL — e.g. `https://docs.kausate.com/api-reference/async-sync.md`
- **Doc index for AI tools**: <https://docs.kausate.com/llms.txt>
- **Changelog**: <https://docs.kausate.com/changelog>
- **Canonical source for this skill**: <https://github.com/kausate/agents>

If anything in this skill conflicts with `openapi.json`, the OpenAPI spec wins.
