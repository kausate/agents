# Endpoint inventory (version `2026-05-01`)

The full source of truth is `https://api.kausate.com/openapi.json`; this is the map an integrator usually needs.

## Search & discovery

| Endpoint | Purpose | Mode | Latency |
|---|---|---|---|
| `GET /v2/companies/search/autocomplete` | Type-ahead during form input | sync | < 100 ms |
| `POST /v2/companies/search` | Live registry search by name / number / advanced query | **async by default**, sync available at `/search/sync` | 2–10 s live |
| `GET /v2/companies/search/person` | Find companies by legal-rep name + DOB | sync | 1–5 s |
| `POST /v2/companies/prefill` | Form-prefill from a known `kausateId`. Index hit + live fallback. Best onboarding UX. | always sync | < 200 ms typical |

Search request body fields: `companyName`, `companyNumber`, `advancedQuery` (jurisdiction-specific structured query), `jurisdictionCode`, `customerReference`. Use `advancedQuery` when you have native registry numbers — it avoids fuzzy matching.

## Translating a native registry ID to a `kausateId`

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

## Company data (async by default, sync available)

| Endpoint | Purpose |
|---|---|
| `POST /v2/companies/report` | Full company report from the registry (legal name, addresses, identifiers, capital, legal reps, shareholders) |
| `POST /v2/companies/finance` | Financials / annual accounts when available |
| `POST /v2/companies/ubo` | Ultimate Beneficial Owners |
| `POST /v2/companies/shareholder-graph` | Multi-level ownership graph. Configurable `maxDepth` (1–7), `enriched` (governance enrichment), `retrievalTimeout` (minutes). Multi-level traversal can take minutes — async is the only sensible mode in production even though `/shareholder-graph/sync` exists. |
| `POST /v2/companies/documents/list` | List the documents available for a company |
| `POST /v2/companies/documents` | Retrieve a specific document (binary served via pre-signed URL in the result) |

All of the above accept the same standard body fields:

- `kausateId` (required) — Kausate's company identifier (`co_{jurisdiction}_{id}`)
- `customerReference` (optional) — your per-request correlation id (see `customer-correlation.md`)
- `bypassCache` (optional, where supported) — skip same-day dedup, force a fresh registry call
- Plus the workflow-specific fields above (e.g. `maxDepth` on shareholder-graph)

For each async endpoint there's a matching sync variant at `/{family}/sync` and a polling endpoint at `GET /v2/companies/{family}/{orderId}` (see `async-webhooks.md`).

## Webhooks, monitors, analytics, jurisdictions

| Endpoint | Purpose |
|---|---|
| `POST /v2/webhooks` / `GET` / `PUT /{id}` / `DELETE /{id}` | Manage webhook subscriptions |
| `POST /v2/monitors` / `GET` / `GET /{id}` / `DELETE /{id}` | Monitor a company on a cron schedule, fire webhook on detected changes |
| `GET /v2/analytics/summary` | Usage / cost rollup, filterable by date range, workflowType, sku, customerId |
| `GET /v2/analytics/timeseries` | Same metrics over time |
| `GET /v2/analytics/breakdowns` | Pivot by jurisdiction, sku, customerId |
| `GET /v2/platform/jurisdictions` | Capability matrix per jurisdiction (which products are supported, what document types exist) |
