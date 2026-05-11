# Monitors — change detection on a schedule

A **monitor** watches one company across one or more **sources** and emits **events** when it detects a change. Each event is classified into a generic, jurisdiction-agnostic taxonomy (`event_code` + `category`) so your routing logic doesn't need a per-jurisdiction mapping table. Per-event billing: 1 credit (`AAKMON`) per scheduled tick of a per-company source, plus 1 credit (`AAKEVT`) per delivered webhook event (refunded if every retry fails).

> **Two distinct webhook channels.** The org-wide `POST /v2/webhooks` subscription (`async-webhooks.md`) delivers **async order completions** (`ExecutionResponse` payloads — camelCase, `result.type` discriminator). Monitor change events are delivered to the **per-monitor `webhookUrl`** you set on `POST /v2/monitors` (`monitor.change_detected` payload — **snake_case**, completely different shape). Don't reuse one handler for both — switch on the path your endpoint receives or use separate URLs.

## Source modes

| Mode | When the source ticks | Examples |
|---|---|---|
| `per_company_pull` | Driven by your monitor's `scheduleCron`. Calls a registry capability and diffs the result against the last observation. | `company_report`, `shareholder_graph` |
| `global_feed` | Runs centrally whenever the upstream publishes; cron is **ignored**. Each item is matched to companies in our index. | `de_insolvenzbekanntmachungen.feed` (German §9 InsO portal) |

## Discover the sources that apply to a company

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

## Categories and event codes

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

## Create a monitor

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

## Webhook payload (`monitor.change_detected`)

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

## Receiver requirements

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

## List + replay events

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

## Auto-deactivation

By default, any event in the `disappeared` category flips the monitor's `is_active` to `false` and writes a `deactivationReason`. The intent is to stop ticking (and stop billing `AAKMON`) for a company that no longer exists. The monitor row is preserved — you can still list its events and replay deliveries — it just doesn't run new ticks.

To opt out entirely, send `"autoDeactivateCategories": []` at create time. To extend (e.g. also stop on insolvency), send the categories you want, e.g. `["disappeared", "status"]`.

## Billing

| SKU | When charged | Refund? |
|---|---|---|
| `AAKMON` | Each tick of a per-company source (5 credits per scheduled run) | No — the registry call already happened |
| `AAKEVT` | Each delivered event (1 credit) | Yes — refunded if every webhook retry fails |

Track usage like any other SKU via `/v2/analytics/breakdowns?groupBy=sku`.

## Common errors

| Status | Reason | Fix |
|---|---|---|
| `404 Company not found` | The `kausateId` doesn't exist in our index | Translate from a native id via `/v2/companies/search` first |
| `422 Unknown monitoring source` | Typo or fabricated source name | Discover via `/v2/events/sources` or the per-company variant |
| `422 Source not applicable to company` | Source is for a different jurisdiction or the company's integration doesn't expose the required capability | List valid sources via `/v2/monitors/sources?kausateId=…` |
| `422 Invalid cron expression` | Cron syntax | Use a valid 5-field cron (e.g. `0 9 * * *`) |
| `422 eventCodesFilter value not in enum` | Stale or misspelled event code | Fetch current codes from `/v2/events/taxonomy` |
