# Migrating from Moody's to Kausate

"Moody's KYC" is a brand-roll-up of **three different products** with three different APIs and three different mental models. The migration that's right for you depends entirely on which one your code calls. Misroute and you'll port the wrong fields.

## Identify your Moody's flavor

| If your code talks to … | You're on … | Read section |
|---|---|---|
| `api.kompany.com/api/v1/...` | **kompany v1** (EOL'd 2024-12-31, no longer operational) | **Part A** |
| `api.kompany.com/api/v2/...` or anything on `app.kompany.com` / `developer.kompany.com` | **kompany v2** | **Part A** |
| `webservices.bvdinfo.com/v1.3/orbis4/remoteaccess.asmx?WSDL` (SOAP) | **Orbis Remote Access (SOAP)** | **Part B** |
| `api.bvdinfo.com/v1/compliancecatalyst4/...` or the Orbis REST Postman workspace | **BvD REST / Compliance Catalyst** | **Part B** |
| `help.maxsight.com` / `developer.maxsight.com` / `api.{us,eu,ae}.maxsight.com/4.0/...` | **Maxsight** (workflow SaaS, formerly Passfort) | **Part C** |

If you talk to more than one, you have more than one migration. The most common combo is "Maxsight UI for compliance ops, kompany API behind it for registry data" — see Part C for how to split that.

> **kompany v1 is dead.** Moody's retired `api.kompany.com/api/v1` on **2024-12-31**: "no longer be operational after the specified date, including the discontinuation of updates, bug fixes, and customer support." If your code is still on v1 paths in 2026, it's been failing for a year — you're not migrating, you're triaging. Skip the v1 → v2 step and jump straight to Kausate.

---

## Mental model — what's actually different

The three Moody's products are not three versions of the same thing. They sit at different points in the KYB stack:

1. **kompany** is a **live-registry API**. You hit an endpoint, kompany makes an outbound call to the source registry (Handelsregister, Companies House, INPI, KVK, …), parses the result, and returns it. Architecturally close to Kausate. Migration is mostly endpoint renaming + payload shape diffs + status-string remapping.

2. **Orbis** is a **pre-ingested database**. BvD spiders 170+ sources, normalises and matches the records, and stores them. You query a cache, not a registry. Most fields refresh **monthly or twice yearly** for institutional extracts; sanctions/PEPs (the Grid) refresh continuously. Customers who came to Orbis expecting "real-time registry" were already on the wrong tool — switching to Kausate gives them what they actually wanted (live calls), at the cost of millisecond-DB-lookup latency vs seconds-to-minutes-per-call.

3. **Maxsight** is a **workflow SaaS on top of providers**. It calls kompany or Orbis under the hood for registry data, calls Grid / Dow Jones / Refinitiv / GBG for screening and document verification, and adds a Profile state machine, review queues, multi-jurisdiction policy packs, and an analyst UI. Maxsight itself isn't a data API — it's an orchestrator. Migrating "off Maxsight" means rebuilding the workflow in your own product, with Kausate filling the registry-data slot.

How each maps to Kausate's live-registry model:

| Source product | Mapping to Kausate |
|---|---|
| kompany | One-to-one for most capabilities. Endpoint shape and status names change; the data primitives don't. |
| Orbis | Architectural shift — you trade "fast DB hit on stale data" for "live registry pull with real-world latency." You'll need a different home for M&A / industry ratios / peer benchmarks / credit ratings — Kausate doesn't carry those. |
| Maxsight | The workflow layer is yours to rebuild. Kausate replaces the registry-data slot; PEP/sanctions/document-verification stays with a separate vendor (or with Maxsight itself, via its API). |

Three deltas common to all three migrations:

1. **Webhooks for order completion.** Kausate posts an `ExecutionResponse` to a single org-wide webhook subscription. kompany has its per-order Product Notifier; Orbis has none (you poll or block on the SOAP session); Maxsight has its own `CHECK_COMPLETED` webhook channel. See `../async-webhooks.md`.
2. **Stable `kausateId`.** Kausate mints `co_{jurisdiction}_{opaque}` once per company. kompany uses a 32-char uppercase hex (`id` in v1, `kompanyId` in v2); Orbis uses `BvDEPID`. Translate once via `POST /v2/companies/search`, cache the mapping.
3. **Date-pinned API version.** `Kausate-Version: 2026-05-01` on every request. No more "we deprecated v1 with three months' notice" — see `../auth-versioning.md`.

---

# Part A — kompany → Kausate

This is by far the most common Moody's migration. Read it linearly: A.1 is the v1→v2 step you may not have finished, A.2 onward is the per-feature map to Kausate. If you're already fully on v2 paths, skim A.1 for the renames and start at A.2.

## A.1 — v1 → v2 breaking changes you may not have done yet

Many readers are stuck mid-migration: they ported some endpoints when v1 was deprecated, then production worked, then v1 was finally turned off and the rest broke. The whole shape changed.

| What changed | v1 | v2 |
|---|---|---|
| Calling convention | Path-positional GET (`/api/v1/{object}/{method}/{p1}/{p2}/...`) — every parameter in the URL | POST with JSON body (`POST /api/v2/{object}/{method}` + `{"field": "value"}`) |
| Identifier field name | `id` (32-char uppercase hex) | `kompanyId` (same hex, renamed field) |
| Dataset name | `full` | `index` |
| System endpoints | `/system/countries` + `/system/health` + `/system/priceList` + `/system/catalog` (4 endpoints) | Collapsed into single `GET /api/v2/system/coverage?countryCode=...&countryConnectionType=...` (2024-09-25) |
| Auth header | `user_key: $KEY` | `user_key: $KEY` (unchanged) |
| v1 retirement | — | EOL **2024-12-31** — updates, bug fixes, and support all discontinued |

The four `/system/*` endpoints were decommissioned **inside v2** on 2024-09-25, replaced by:

```
GET https://api.kompany.com/api/v2/system/coverage
   ?countryCode=AT|US-DE
   &countryConnectionType=L|RU|BC|D|NC
```

Connection types: `L` = Live, `RU` = Live with Regular Updates, `BC` = Business Concierge (human-mediated only), `D` = Discontinued, `NC` = Not Covered Yet.

### `full` → `index` and the 11 disabled jurisdictions

The `full` dataset was renamed to `index` in v2. After the rename, `index` was **disabled in 11 jurisdictions** — in those, you have to use `refresh` (live registry pull) or `super` instead:

```
HU, US-TX, US-CA, ES, FI, GI, JE, LT, MT, SI, CA
```

Code-grep for the literal string `"datasetName":"full"` and `"id":` in dataset POST bodies — if you find any, your v2 migration is incomplete. (After this step you'll be migrating from v2 to Kausate, so don't over-invest — see A.2.)

### Don't migrate v1 → v2 → Kausate. Migrate v1 → Kausate directly.

If you're still on v1 in 2026, the rational path is **skip v2 entirely**. Migrating v1 → v2 → Kausate is two migrations; v1 → Kausate is one. v2 is also vendor-controlled in the same way v1 was — there's nothing to stop Moody's from doing it again. Kausate's date-based versioning header (see `../auth-versioning.md`) is the structural fix.

## A.2 — Endpoint mapping (v1 + v2 → Kausate)

| Capability | kompany v1 | kompany v2 | Kausate (`2026-05-01`) |
|---|---|---|---|
| Index search (cached) | `GET /api/v1/company/search/{country}/{name}` | `POST /api/v2/company/search` with `{"country","name"}` | `GET /v2/companies/search/autocomplete?name=...&jurisdictionCode=...` |
| Deep search (live registry) | `GET /api/v1/company/deepsearch/{name\|number}/{country}/{q}` | `POST /api/v2/company/deepsearch` | `POST /v2/companies/search` (async) or `/v2/companies/search/sync` |
| Dataset: `mini` | `GET /api/v1/company/{id}/mini` | `POST /api/v2/company/dataset` with `{"datasetName":"mini","kompanyId":"..."}` | `POST /v2/companies/prefill` (sync, < 200 ms) |
| Dataset: `master` | `GET /api/v1/company/{id}/master` | (legacy) | `POST /v2/companies/prefill` or shallow `POST /v2/companies/report` |
| Dataset: `full` (v1) / `index` (v2) | `GET /api/v1/company/{id}/full` | `POST /api/v2/company/dataset` with `{"datasetName":"index","kompanyId":"..."}` | `POST /v2/companies/prefill` (index hit + live fallback) — no exact equivalent; `prefill` is the cached fast tier |
| Dataset: `refresh` | `GET /api/v1/company/{id}/refresh` | `POST /api/v2/company/dataset` with `{"datasetName":"refresh","kompanyId":"..."}` | `POST /v2/companies/report` (always live) |
| Dataset: `super` (2024+) | — | `POST /api/v2/company/dataset` with `{"datasetName":"super","kompanyId":"..."}` | `POST /v2/companies/report` with `{"bypassCache":true}` (forces a fresh registry call) |
| Product availability | `GET /api/v1/product/availability/{sku}/{subjectId}` | `POST /api/v2/product/availability` | `POST /v2/companies/documents/list` (the listing is the availability) |
| Place product order | `POST /api/v1/product/order/{sku}/{subjectId}` | `POST /api/v2/product/order` | `POST /v2/companies/documents` with `kausateDocumentId` from list |
| Order status | `GET /api/v1/product/status/{orderId}` | `POST /api/v2/product/status` | Webhook delivery; polling fallback `GET /v2/companies/documents/{orderId}` |
| Retrieve artifact | `GET /api/v1/product/{orderId}` (byte stream) | `POST /api/v2/product/retrieve` | `result.downloadLink` (pre-signed S3 URL) on webhook payload |
| Product Notifier (per-order webhook) | `POST /api/v1/product/notifier/{orderId}/{type}/{uri}` | `POST /api/v2/product/notifier` | **Single org-wide** `POST /v2/webhooks` subscription; switch on `result.type` |
| Concierge | `POST /api/v1/product/order/concierge` | `POST /api/v2/product/order/concierge` | No direct equivalent; place the closest standard order or contact support |
| UBO discovery | `POST /api/v1/ubodiscovery/order/{subjectId}` | `POST /api/v2/ubodiscovery/order` | `POST /v2/companies/ubo` (single layer) or `POST /v2/companies/shareholder-graph` (multi-level) |
| UBO update (paused/awaiting input) | `POST /api/v1/ubodiscovery/update/{discoveryOrderId}` | `POST /api/v2/ubodiscovery/update` | Not exposed — Kausate handles paused-discovery internally; you receive the final webhook |
| UBO result fetch | `GET /api/v1/ubodiscovery/{discoveryOrderId}` | `POST /api/v2/ubodiscovery/result` | Webhook delivery (`result.type: uboReport` / `shareholderGraph`); polling fallback `GET /v2/companies/ubo/{orderId}` |
| Monitoring register | `POST /api/v1/monitoring/register/{subjectId}/{changeType}` | `POST /api/v2/monitoring/register` | `POST /v2/monitors` with `sources` + `categoriesFilter` |
| Monitoring list | `GET /api/v1/monitoring/list[/{subjectId}]` | `POST /api/v2/monitoring/list` | `GET /v2/monitors` |
| Monitoring change types | `GET /api/v1/monitoring/changetypes` | `POST /api/v2/monitoring/changetypes` | `GET /v2/events/taxonomy` (returns `categories` + `eventCodes`) |
| Monitoring unregister | `POST /api/v1/monitoring/unregister/{monitorId}` | `POST /api/v2/monitoring/unregister` | `DELETE /v2/monitors/{monitorId}` |
| Announcements | `GET /api/v1/announcements/check/{subjectId}` + `/notify/register/{subjectId}/{uri}` | `POST /api/v2/announcements/check` + `/notify/register` | `POST /v2/monitors` with a feed-mode source (e.g. `de_insolvenzbekanntmachungen.feed`) |
| PEP / sanctions / adverse media | `POST /api/v1/pepsanction/check` (singular) | `POST /api/v2/pepsanction/check` | **Not in Kausate** — separate-vendor concern (Sanctions360, Refinitiv, ComplyAdvantage, …) |
| PEP/sanctions monitoring | `POST /api/v1/pepsanction/monitoring/register/{reportId}` | `POST /api/v2/pepsanction/monitoring/register` | **Not in Kausate** |
| Tax / VAT verify | `GET /api/v1/euvat/verify/basic/{vat}` (+ comprehensive / level2) | `POST /api/v2/euvat/verify/{tier}` | **Not in Kausate** — call VIES directly or use a dedicated VAT vendor |
| TIN / EIN / IBAN | `GET /api/v1/{tin,ein,iban}/verify/{value}` | `POST /api/v2/...` | **Not in Kausate** — separate-vendor concern |
| Stock-exchange listing | `GET /api/v1/stockexchange/isin/lei/{lei}` + `/listing/{isin}` | `POST /api/v2/stockexchange/...` | **Not in Kausate** — Kausate is registry data, not market data |
| System discovery | `/system/{countries,health,priceList,catalog}` (v1, 4 endpoints) → consolidated `/system/coverage` (v2, 2024-09-25) | `GET /api/v2/system/coverage?countryCode=...` | `GET /v2/platform/jurisdictions` |

The full inventory of Kausate endpoints is in `../endpoints.md`. The async lifecycle and webhook receiver pattern are in `../async-webhooks.md` — don't re-implement them here.

## A.3 — Auth diff

```diff
- user_key: {API_KEY}              # kompany v1 + v2 — both use the same header
+ X-API-Key: $KAUSATE_API_KEY
+ Kausate-Version: 2026-05-01      # pin the version on every request
```

The kompany `user_key` header was unchanged across v1 and v2; only Kausate has the version pin. See `../auth-versioning.md` for why pinning matters and how to generate a typed client from `openapi.json`.

## A.4 — Identifier translation: kompany hex `id` / `kompanyId` → `kausateId`

kompany's identifier is a 32-character uppercase hex string, derived internally (not published) from the tuple `(country, registrationNumber)`. In v1 it's the field `id`; in v2 it's renamed to `kompanyId`. Same value.

```json
// kompany index-search response
{
  "id": "77A5AD48BBB33E5E3A397FFC920C3885",
  "country": "AT",
  "registrationNumber": "375714x",
  "name": "360kompany AG",
  "requestTime": 1627645107,
  "lastUpdate": 1627511705
}
```

> The `x` suffix in `"375714x"` is intentional — kompany **does not normalise** registration numbers. You pass exactly what the source registry uses, including any case-sensitive letters. Same applies to Kausate `companyNumber`.

Translate to `kausateId` once, cache the mapping:

```bash
# kompany style (v1): GET /api/v1/company/search/AT/360kompany
# Kausate equivalent — note jurisdiction is lowercase ISO:
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyName":"360kompany","jurisdictionCode":"at"}'
# → result.searchResults[0].kausateId  = "co_at_..."
```

```python
# Python (httpx) — translate native id → kausateId, with a per-process cache
import httpx
from functools import lru_cache

@lru_cache(maxsize=10_000)
def kompany_id_to_kausate_id(country: str, reg_number: str) -> str:
    r = httpx.post(
        "https://api.kausate.com/v2/companies/search/sync",
        headers={
            "X-API-Key": KAUSATE_API_KEY,
            "Kausate-Version": "2026-05-01",
        },
        json={
            "companyNumber": reg_number,
            "jurisdictionCode": country.lower(),  # "AT" -> "at"
        },
        timeout=15.0,
    )
    r.raise_for_status()
    return r.json()["result"]["searchResults"][0]["kausateId"]
```

Persist `(kompany_id, kausate_id)` in your database. The `kausateId` is stable — no need to re-translate on every call. See `../endpoints.md` for more on the translation step.

## A.5 — Per-feature subsections

### Index search

**kompany:**

```
GET /api/v1/company/search/AT/360kompany           # v1
POST /api/v2/company/search                        # v2
Body: {"country":"AT","name":"360kompany","limit":10}
```

Country code traps:
- **`UK` is the kompany country code for the United Kingdom**, covering England, Wales, Scotland, and Northern Ireland. **ISO is `GB`.** Kausate uses lowercase ISO (`gb`).
- **`US-{STATE}` for United States states**, e.g. `US-NY`, `US-DE`, `US-CA`, `US-TX`. Kausate uses `us_de`, `us_ny`, etc. (lowercase, underscore-separated).
- All other jurisdictions: ISO-3166 alpha-2, uppercase in kompany, **lowercase in Kausate** (`de`, `fr`, `nl`, …).

**Kausate (autocomplete — type-ahead UX):**

```bash
curl "https://api.kausate.com/v2/companies/search/autocomplete?name=360kompany&jurisdictionCode=at" \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"
```

Sub-100 ms; sync. Use for form input fields. For canonical "given a name, find the company" use the deep search below.

### Deep search (live registry)

**kompany (v1, path-positional GET):**

```
GET /api/v1/company/deepsearch/name/AT/360kompany
GET /api/v1/company/deepsearch/number/AT/375714x
```

**kompany (v2, POST):**

```
POST /api/v2/company/deepsearch
Body: {"searchType":"name|number","country":"AT","q":"360kompany"}
```

2–4 s latency (live registry call), per-jurisdiction gov / registry fees may apply.

**Kausate:**

```bash
# Async — production pattern
curl -X POST https://api.kausate.com/v2/companies/search \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyName":"360kompany","jurisdictionCode":"at","customerReference":"search-001"}'
# → {"orderId":"ord_...", "status":"running"} — result arrives on webhook (result.type: "liveSearch")

# Sync — prototyping only, hard 300 s timeout
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyName":"360kompany","jurisdictionCode":"at"}'
```

### Datasets (`mini` / `master` / `full` / `refresh` / `super` / `index`)

The kompany dataset zoo flattens into three Kausate calls. The mapping is about freshness, not field shape:

| kompany dataset | What kompany returns | Latency | Kausate equivalent |
|---|---|---|---|
| `mini` | `id, country, registrationNumber, name` only | fastest (cached) | `POST /v2/companies/prefill` (returns the same identification primitives plus a few more) |
| `master` (v1) | `mini` + address fields | fast (cached) | `POST /v2/companies/prefill` |
| `full` (v1) → `index` (v2) | jurisdiction-dependent canonical record from kompany's index | variable (cached) | `POST /v2/companies/prefill` — closest equivalent. No direct "give me the cached record without a live fallback" tier in Kausate. |
| `refresh` | forced re-fetch from the primary source registry | variable (live) | `POST /v2/companies/report` (always live by default) |
| `super` (2024+) | jurisdiction-dependent — officers + shareholders where the source register exposes them; most-fresh | variable (live) | `POST /v2/companies/report` with `{"bypassCache": true}` |

The `super` dataset shipped on 2025-04-16 as the most-fresh tier; `refresh` had a similar intent. Both map cleanly to `POST /v2/companies/report` — the only knob you'll touch is `bypassCache` (forces same-day-dedup skip, see `../status-and-dedup.md`).

**`index` is disabled in 11 jurisdictions** post-rename:

```
HU, US-TX, US-CA, ES, FI, GI, JE, LT, MT, SI, CA
```

If your code falls back to `refresh` in those jurisdictions, on Kausate that's just `POST /v2/companies/report` — same call, no special-casing.

### Product orders (certified extracts, filings, financials)

kompany's product-order surface is a four-call dance: availability → order → status → retrieve. Kausate collapses it to a two-step async flow with a single webhook delivery.

**kompany (v1):**

```bash
# 1. Check availability of a SKU for a subject
curl -H "user_key: $KEY" \
  https://api.kompany.com/api/v1/product/availability/DOCOFCHUK/$SUBJECT_ID
# → {"available": true, "hasOptions": false, ...}

# 2. Place the order
curl -X POST -H "user_key: $KEY" \
  https://api.kompany.com/api/v1/product/order/DOCOFCHUK/$SUBJECT_ID
# → {"identity":"4B5543C654A39D5CCC7197F0EB74F0DF","sku":"DOCOFCHUK","status":"PROCESSING","ordered":"1590492217"}

# 3. Poll status — kompany doc says "loop every 60 seconds until completed"
curl -H "user_key: $KEY" \
  https://api.kompany.com/api/v1/product/status/$ORDER_ID
# → {"status": "PROCESSING|COMPLETED|FAILED|CANCELLED"}

# 4. Retrieve the artifact (byte stream with original Content-Type)
curl -H "user_key: $KEY" \
  https://api.kompany.com/api/v1/product/$ORDER_ID > extract.pdf
```

**Kausate (v2026-05-01):**

```bash
# 1. List documents available for a company (replaces availability + sku enumeration)
curl -X POST https://api.kausate.com/v2/companies/documents/list \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"kausateId":"co_gb_...","customerReference":"docs-list-1"}'
# → {"orderId":"ord_...","status":"running"}
# → webhook fires with result.type: "listDocuments" and a list of {kausateDocumentId, title, documentType, ...}

# 2. Place the document order (using a kausateDocumentId from the list)
curl -X POST https://api.kausate.com/v2/companies/documents \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"kausateId":"co_gb_...","kausateDocumentId":"doc_...","customerReference":"docs-fetch-1"}'
# → {"orderId":"ord_...","status":"running"}
# → webhook fires with result.type: "document" carrying a pre-signed S3 URL (downloadLink)
```

The full document workflow (with payload shapes, expiry semantics, and caching guidance) is in `../documents.md`. Key differences vs kompany:

- **Two steps, not four.** Kausate's `documents/list` returns the catalog; `documents` retrieves a specific item. The "availability" and "status" calls collapse into the async lifecycle (status is the order's status; webhooks replace polling).
- **No SKU strings to memorise.** kompany SKUs (`DOCOFCHUK`, `DOCOFCRIE`, `ADDAAVKDK`, `ADDOFNYUS`, …) are replaced by `documentType` values returned in the listing (`currentExtract`, `articlesOfAssociation`, `shareholderList`, `chronologicalExtract`, `annualAccounts`, …). You filter the list by `documentType`, you don't construct a SKU from a jurisdiction code.
- **Bytes vs URL.** kompany returns the PDF bytes on `GET /product/{orderId}`. Kausate returns a **pre-signed `downloadLink`** in the webhook `result`; fetch the bytes from that URL (no `X-API-Key` needed; the URL is signed; respect `expiresAt`).
- **kompany's 60-second polling cadence is gone.** Kausate's webhook replaces it. If you must poll, use exponential backoff (5 s start, 1.2× multiplier, 15 s cap, 30 min total) — see `../async-webhooks.md`.

Known kompany SKUs you might find in your codebase, with mapped `documentType`:

| kompany SKU | Jurisdiction | Kausate `documentType` |
|---|---|---|
| `DOCOFCHUK` | UK Companies House extract | `currentExtract` |
| `DOCOFCRIE` | Ireland CRO certified extract | `currentExtract` |
| `ADDAAVKDK` | Denmark Annual Accounts | `annualAccounts` |
| `ADDOFNYUS` | New York Registration Report | `registrationDetails` |
| `CONCIERGE_EXPRESS` / `CONCIERGE_STANDARD` | Human-mediated concierge | (no equivalent — contact Kausate support) |

The full SKU list per jurisdiction was gated behind `developer.kompany.com` and returned via v2 `/system/coverage`. Kausate's per-jurisdiction capability matrix is at `GET /v2/platform/jurisdictions`.

### UBO Discovery

**kompany:**

```
POST /api/v1/ubodiscovery/order/{subjectId}       # POST kicks off the async order
POST /api/v1/ubodiscovery/update/{discoveryOrderId}  # only when paused (human input needed)
GET  /api/v1/ubodiscovery/{discoveryOrderId}      # fetch the result once done
```

Configurable parameters (threshold percentage, max depth) were marketing copy with no public technical reference — they lived behind `developer.kompany.com`. Coverage was 30 countries in early 2024, then 31 with Greece in mid-2024. Report delivered in two formats: structured JSON **and** PDF.

**Kausate:**

```bash
# Single-layer UBO (immediate beneficial owners)
curl -X POST https://api.kausate.com/v2/companies/ubo \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"kausateId":"co_de_...","customerReference":"ubo-001"}'

# Multi-level ownership graph
curl -X POST https://api.kausate.com/v2/companies/shareholder-graph \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId":"co_de_...",
    "maxDepth": 5,
    "enriched": true,
    "retrievalTimeout": 30,
    "customerReference":"graph-001"
  }'
```

Parameter shift to be aware of:

| kompany | Kausate |
|---|---|
| Threshold percentage (gated; private) | No explicit threshold — graph returns full ownership down to `maxDepth`; you filter client-side if you want a UBO threshold |
| Max depth (gated; private) | `maxDepth` 1–7 on `shareholder-graph` (explicit, public) |
| Two formats: JSON + PDF | JSON only via webhook; if you need a PDF, render server-side from the JSON or order an articles-of-association / shareholder-list document via `/v2/companies/documents` |
| Paused / update flow for human-input layers (e.g. trust intermediaries) | Not exposed to the caller — Kausate handles paused-discovery internally; you receive the final webhook delivery |

### Monitoring

**kompany:**

```
POST /api/v1/monitoring/register/{subjectId}/{changeType}
POST /api/v1/monitoring/unregister/{monitorId}
GET  /api/v1/monitoring/list/{subjectId}
GET  /api/v1/monitoring/list
GET  /api/v1/monitoring/changetypes   # the enum of event types
```

Monitoring change notifications were delivered through the **Product Notifier** webhook — the same per-order webhook subscription mechanism used for product orders. The change-type enum was gated; `GET /monitoring/changetypes` was the discoverable list (officer-change, address-change, financials-filed, status-change, etc.).

**Kausate:**

```bash
# 1. Discover which sources apply to this company (sources are jurisdiction-gated)
curl -G https://api.kausate.com/v2/monitors/sources \
  --data-urlencode "kausateId=co_de_..." \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"

# 2. Create a monitor with one or more sources + a per-monitor webhookUrl
curl -X POST https://api.kausate.com/v2/monitors \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId":"co_de_...",
    "sources":["company_report","shareholder_graph","de_insolvenzbekanntmachungen.feed"],
    "scheduleCron":"0 9 * * *",
    "webhookUrl":"https://your-app.example.com/webhooks/kausate-monitors",
    "categoriesFilter":["status","ownership","disappeared"],
    "autoDeactivateCategories":["disappeared"]
  }'

# 3. The event taxonomy is discoverable (replaces /monitoring/changetypes)
curl https://api.kausate.com/v2/events/taxonomy \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01"
```

Migration footguns specific to monitoring:

- **Product Notifier was unified with product-order webhooks.** Kausate splits them: `POST /v2/webhooks` for order completions (`ExecutionResponse` payload, camelCase), per-monitor `webhookUrl` for change events (`monitor.change_detected` payload, **snake_case**, completely different shape). Don't reuse one handler — host them at different paths.
- **kompany's `changeType` is a single string per registration.** Kausate's `sources` is a list (you can monitor `company_report` + `shareholder_graph` + a feed source on one monitor), filtered by `categoriesFilter` / `eventCodesFilter`. Map your single-changeType kompany registrations to single-element `sources` lists if you want behavioural parity.
- **Auto-deactivation has no kompany equivalent.** By default, any `disappeared` event flips `is_active=false` and stops billing. Pass `"autoDeactivateCategories": []` to opt out.

The full monitor surface (sources, categories, taxonomy, billing, replay) is in `../monitors.md`.

### Announcements (statutory gazette / registry announcements)

**kompany:**

```
GET  /api/v1/announcements/check/{subjectId}
GET  /api/v1/announcements/get/{subjectId}
POST /api/v1/announcements/notify/register/{subjectId}/{uri}
POST /api/v1/announcements/notify/unregister/{notifierId}
GET  /api/v1/announcements/notify/list/{subjectId}
GET  /api/v1/announcements/notify/list
```

Country support was gated; sales contact was the only way to confirm. Same **tilde-for-slash encoding** rule as Product Notifier (see footguns below).

**Kausate:** Announcements are folded into the monitor model as **feed-mode sources**. For German `Insolvenzbekanntmachungen` (§9 InsO portal):

```bash
curl -X POST https://api.kausate.com/v2/monitors \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId":"co_de_...",
    "sources":["de_insolvenzbekanntmachungen.feed"],
    "scheduleCron":"0 0 * * *",
    "webhookUrl":"https://your-app.example.com/webhooks/kausate-monitors"
  }'
```

Feed-mode sources are global feeds matched server-side to companies in the Kausate index. The `scheduleCron` is **ignored** for feeds (they run on upstream cadence) but is still a required field — pass any valid expression. Feed events have `before: null` and `after: <publication payload>` in the webhook (the publication itself is the change). See `../monitors.md`.

### PEP / sanctions / adverse media

**kompany:**

```
POST /api/v1/pepsanction/check         # note: pepsanction (singular)
POST /api/v1/pepsanction/monitoring/register/{reportId}
POST /api/v1/pepsanction/monitoring/update/{monitoringId}
POST /api/v1/pepsanction/monitoring/unregister/{monitoringId}
```

**Watch the spelling**: the path is `pepsanction` (singular, no `s`) — a frequent typo source when migrating. Underlying data source is named "Sanctions360" in Maxsight's stack.

**Kausate:** Not provided. Kausate is registry data — PEP, sanctions, and adverse-media screening are a separate-vendor concern. Common alternatives: Refinitiv World-Check, Dow Jones Risk and Compliance, ComplyAdvantage, Moody's own Grid (you can keep kompany/Grid for screening even while migrating registry data to Kausate). If you're on Maxsight, Maxsight's `INDIVIDUAL_ASSESS_MEDIA_AND_POLITICAL_AND_SANCTIONS_EXPOSURE` / `COMPANY_ASSESS_MEDIA_AND_SANCTIONS_EXPOSURE` tasks stay where they are — see Part C.

### Tax / VAT / TIN / EIN / IBAN

**kompany:** broad surface — `/api/v1/taxid/verify`, `/euvat/verify/{basic|comprehensive|level2}`, `/tin/{lookup|verify}`, `/ein/{validate|verify|get}`, `/iban/verify`, `/ptnif/verify` (Portugal NIF). EU VAT integrates VIES; "level2" tier returns an audited certificate.

**Kausate:** Not provided. VAT verification is also out of scope. Call VIES directly (`ec.europa.eu/taxation_customs/vies/services/checkVatService` SOAP, plus REST mirrors) or use a dedicated VAT vendor (Vatstack, Vatlayer, etc.). Tax ID verification per jurisdiction is similarly a separate-vendor space.

### Stock-exchange listing

**kompany:** `GET /api/v1/stockexchange/isin/lei/{lei}` + `/listing/{isin}`. **Kausate:** not provided — Kausate is registry data, not market data. Use the LEI registry (GLEIF) and a market-data vendor (FactSet, Refinitiv, Bloomberg) for listing lookups.

### System / capability discovery

**kompany v2:** `GET /api/v2/system/coverage?countryCode=...&countryConnectionType=...` — single endpoint, replaces the four v1 `/system/*` calls. Returns sources, available datasets, available documents (SKUs), search modes, concierge products per jurisdiction.

**Kausate:** `GET /v2/platform/jurisdictions` returns the capability matrix per jurisdiction (which products are supported, what document types exist). Combine with `GET /v2/monitors/sources?kausateId=...` for per-company source discovery (see `../monitors.md`).

## A.6 — Encoding quirks that will silently break your port

These are real artifacts of the kompany API surface, documented in the v1 reference. If you don't account for them, your port will compile, your tests will pass on happy paths, and production will fail in subtle ways.

1. **Tilde-for-slash in callback URI path segments.** v1 paths like `/api/v1/product/notifier/{orderId}/{type}/{uri}` and `/api/v1/announcements/notify/register/{subjectId}/{uri}` put the callback URL **in the path**. Forward slashes in the URL had to be replaced with `~` before passing them in the path; kompany decoded `~` back to `/` server-side before POSTing the callback. **A naive `encodeURIComponent` of the URL breaks the route** — kompany doesn't decode `%2F`, it expects `~`. Kausate uses a JSON body field (`url`) for `POST /v2/webhooks` and `webhookUrl` for `POST /v2/monitors` — drop the tilde-substitution code entirely.

2. **`fiancialData` typo in the Concierge body.** The kompany Concierge endpoint accepts optional booleans `registerData, fiancialData [sic], historicInformation, localInvestigation`. The typo (`fiancial` missing the `n`) is the production spelling. **Don't "correct" it client-side** — the corrected spelling fails server-side validation. If you had a wrapper that normalised the typo before forwarding to kompany, that wrapper does nothing useful for Kausate (Kausate doesn't have a concierge endpoint at all).

3. **`pepsanction` (singular) in v1 paths.** Easy to over-pluralise to `pepsanctions`. Kausate doesn't have this endpoint, but if your shared HTTP client wraps both kompany and Kausate calls, the singular path needs to survive the port.

4. **`UK` is kompany's UK country code, not `GB`.** Covers England, Wales, Scotland, NI. Kausate uses ISO `gb` (lowercase). Code-grep for the literal string `"UK"` and `'UK'` in country-code position when porting.

5. **`US-{STATE}` (uppercase, hyphen) in kompany, `us_{state}` (lowercase, underscore) in Kausate.** Both `US-DE` (Delaware) and `US-CA` (California) become `us_de` / `us_ca` in Kausate.

## A.7 — Migration footguns (kompany-specific)

1. **Country-code casing.** kompany used `UK` (special), `US-{STATE}` (uppercase + hyphen), and uppercase ISO elsewhere. Kausate is **lowercase ISO** everywhere, with US states as `us_xx`. Build a translation function and call it at every API boundary.

2. **`customerReference` vs `customerId` — different semantics from kompany's flat ID.** kompany had `user_key` as the sole tenant identifier; there was no documented "behalf-of" or sub-account header. Customers who served their own sub-customers (compliance platforms like Maxsight) attributed internally and amortised costs. Kausate splits this into two fields:
   - **`customerReference`** — your per-request correlation id (KYC case id, your order id). Echoes back in webhook payload. Body field.
   - **`customerId`** — your end-customer / tenant / sub-account id. Echoes back in webhook payload **and** drives analytics breakdowns (`/v2/analytics/breakdowns?groupBy=customerId`). Sent as `X-Customer-Id` header (not body — body submission returns 422 because request models are `extra=forbid`).
   
   See `../customer-correlation.md`.

3. **Product Notifier ≠ unified Kausate webhook.** kompany registered a Product Notifier **per order** (`POST /api/v1/product/notifier/{orderId}/{type}/{uri}`). Kausate has **one org-wide subscription** (`POST /v2/webhooks`) that receives every order completion for your org. Switch on `result.type` in your handler — see `../async-webhooks.md`. Don't try to "subscribe per order" — that pattern doesn't exist in Kausate.

4. **Order webhook ≠ monitor webhook.** Two distinct channels with different payloads (camelCase `ExecutionResponse` vs snake_case `monitor.change_detected`). Host them on different paths. See `../monitors.md`.

5. **Status-string mismatch.** kompany strings → Kausate enum — see the table in §Status mapping below.

6. **`id` vs `kompanyId` in dataset POSTs.** If your code is partway through v1 → v2 and uses both, the field rename is silent — a wrong field name returns a 4xx with a generic message. Code-grep for `"id":` adjacent to `"datasetName":` and replace with `"kompanyId":`.

7. **`full` → `index` rename in `datasetName`.** Same silent-failure shape. Code-grep for `"datasetName":"full"`.

8. **`index` is disabled in 11 jurisdictions.** In `HU, US-TX, US-CA, ES, FI, GI, JE, LT, MT, SI, CA`, you must use `refresh` or `super`. On Kausate, all of these collapse to `POST /v2/companies/report` — but if you had jurisdiction-specific code branching on dataset name, audit it before porting.

9. **kompany's 60-second polling cadence was normative, not advisory.** Polling faster than 60 s on `GET /api/v1/product/status/{orderId}` was historically rate-limited. Kausate's webhook replaces polling; if you fall back to polling, use the 5 s → 1.2× → 15 s cap schedule from `../async-webhooks.md`, not kompany's 60 s loop.

---

# Part B — Orbis (BvD) → Kausate

The hardest of the three Moody's migrations. Orbis is not a registry API — it's a **pre-ingested database** with a SOAP-based query language and a parallel REST surface (Compliance Catalyst). Moving to Kausate is a category shift, not an endpoint rename.

## B.1 — What's different architecturally

| Orbis | Kausate |
|---|---|
| Pre-ingested DB — 170+ sources spidered, normalised, matched, stored | Live registry pull — Kausate calls the source registry on every order |
| Twice-yearly institutional extracts; historical extracts annually; some fields weekly; Grid (sanctions/PEPs) continuously | Live as of the registry's last update — typically same-day |
| Sub-second DB lookups | Seconds-to-minutes per registry call (real-world async latency) |
| SOAP envelopes + session tokens (Open / CreateQuery2 / GetData2 / Close) — stateful | REST + `X-API-Key` per request — stateless |
| Seat-license subscription (5–6 figure annual) | Per-call credit consumption |
| `BvDEPID` (extended entity identifier, e.g. `DE12345678`) — packs `{country_prefix}{national_reg_number}` | `kausateId = co_{jurisdiction}_{opaque}` — stable, opaque |
| Bulk-friendly (`SearchByBvDIds` happily takes 10k IDs; `POST /Companies` returns up to 4 000 results per query) | Per-company at HTTP-request granularity; bulk = fan out + collect via webhook |
| Aggregated financials, ratios, M&A, peer benchmarks, credit ratings, Grid compliance flags | Raw registry data only — no derived analytics, no aggregated financials |

Customers who came to Orbis expecting "real-time registry" were on the wrong tool from day one. Migrating those customers to Kausate gives them what they actually wanted. Customers who came for the **analytics layer** (M&A, ratios, peer benchmarks) need a different home — Kausate can replace the registry-data half, not the analytics half. The two can coexist.

## B.2 — SOAP endpoint mapping (Orbis Remote Access v1.3)

Operations confirmed via search snippets and the WSDL index page (`https://webservices.bvdinfo.com/v1.3/orbis4/remoteaccess.asmx?WSDL`). Full input/output schema is gated.

| Orbis SOAP operation | Purpose | Kausate equivalent |
|---|---|---|
| `Open` | Open a session — `User` + `Password`; returns a session token | Stateless — `X-API-Key` header per call. **Delete your session-management code entirely.** |
| `Close` | Close a session | (n/a) |
| `CreateQuery` / `CreateQuery2` | Define a search strategy + a list of fields to return | (n/a) — Kausate calls are per-company, not per-saved-query. The "WHERE + SELECT + TOP" model is replaced by: search → translate to `kausateId` → call the specific endpoint that returns the fields you need. |
| `GetData` / `GetData2` | Execute a defined query, return a paged result set | `POST /v2/companies/report` (per `kausateId`) — async |
| `SearchCompanies` | One-shot company search; returns BvD IDs + basic identification | `POST /v2/companies/search` (async) or `/v2/companies/search/sync` |
| `SearchByBvDIds` | Bulk lookup by `BvDEPID` list | **No bulk endpoint.** Fan out per-company asynchronously; collect via webhook. See B.5. |
| `Find` | Single-call match by name + address | `POST /v2/companies/search` with `companyName` + `jurisdictionCode` |
| `ListAvailableData` | Returns the field catalog available to the calling tenant | `GET /v2/platform/jurisdictions` — capability matrix per jurisdiction |

**Session model: gone.** Orbis was stateful — you `Open`d once, reused the session token, paginated `GetData`, then `Close`d. Concurrent sessions were license-controlled. Kausate is stateless: every request carries `X-API-Key` + `Kausate-Version`. There's no session-count limit; per-org concurrency is governed by your credit-bucket and rate limits.

## B.3 — REST endpoint mapping (Compliance Catalyst / Orbis REST)

Two parallel REST surfaces share the same auth model:

- **Compliance Catalyst:** `https://api.bvdinfo.com/v1/compliancecatalyst4/`
- **Orbis REST:** documented in the public Postman workspace `postman.com/orbisrest/orbis-rest-documentation-s-public-workspace`

**Auth (both):** either `ApiToken: <token>` header or `Authorization: Bearer <JWT>`. TLS 1.2 required. **`429 Too Many Requests`** is the canonical concurrency error: "Your account dictates the number of simultaneous calls allowed at any one time, and exceeding this count will result in a 429."

| BvD REST endpoint | Purpose | Kausate equivalent |
|---|---|---|
| `POST /Match` | Match a company by name + country + optional VAT/national number; ranked top-50 | `POST /v2/companies/search` (async) — set `companyName` + `jurisdictionCode` |
| `POST /Companies` (or "plain search") | Exhaustive search — up to **4 000 results**; for batch / due-diligence sweeps | `POST /v2/companies/search` (no built-in 4 000-result page; paginate the search response, or use `GET /v2/companies/search/autocomplete` for type-ahead) |
| `POST /SearchByBvDIds` | Bulk hydrate by `BvDEPID` list | **No bulk equivalent.** Fan out per-company; see B.5 |
| `GET /Metadata/Companies` | List available fields / data points for the tenant | `GET /v2/platform/jurisdictions` |
| `POST /CompanyData` (Compliance Catalyst) | Fetch a configured set of fields for a given `BvDEPID` | `POST /v2/companies/report` (per `kausateId`) |

The body of these calls is a JSON "Query" object wrapping three things: a `WHERE` (filter expression), a `SELECT` (list of data-point identifiers from the Metadata endpoint), and a `TOP n` (paging directive). This is the SOAP "Strategy" model carried into REST. Kausate has no equivalent — instead of selecting fields, you call the endpoint that returns the data primitive you want (`report` / `finance` / `ubo` / `shareholder-graph` / `documents`), and the response shape is determined by the API version you pinned (`Kausate-Version: 2026-05-01`).

**Auth diff:**

```diff
- ApiToken: {TOKEN}                          # BvD REST — single header
- Authorization: Bearer {JWT}                # BvD REST — JWT alternative
+ X-API-Key: $KAUSATE_API_KEY
+ Kausate-Version: 2026-05-01
```

## B.4 — `BvDEPID` → `kausateId`

`BvDEPID` packs `{country_prefix}{national_reg_number}` (e.g. `DE12345678`, `GB04124965`). Strip the country prefix, lowercase it, and translate via search:

```bash
# BvDEPID DE12345678 → kausateId
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyNumber":"12345678","jurisdictionCode":"de"}'
# → result.searchResults[0].kausateId
```

Other Orbis identifiers — `BvD ID` (`bvd_id`), `Orbis ID`, `Cortera Link ID`, `LEI`, `VAT`, `EU VAT`, national number — also flow through `POST /v2/companies/search`. The native-id formats per jurisdiction are in `../identifiers.md`.

## B.5 — Bulk-lookup migration

Orbis is **built for bulk**. `POST /Companies` returns up to 4 000 results; `SearchByBvDIds` accepts a list of IDs; SOAP `GetData2` pages efficiently. Customers run population sweeps that hydrate 10k+ companies in a single workflow.

**Kausate has no bulk endpoint.** The pattern is:

1. Fan out — submit per-company async orders in parallel (Kausate enforces per-org concurrency limits; back off on 429).
2. Receive completions via webhook.
3. Aggregate server-side.

```python
# Python (asyncio + httpx) — bulk fan-out for an Orbis-style population sweep
import asyncio, httpx

KAUSATE_API_KEY = ...

async def submit_report(client, kausate_id, customer_ref):
    r = await client.post(
        "https://api.kausate.com/v2/companies/report",
        headers={
            "X-API-Key": KAUSATE_API_KEY,
            "Kausate-Version": "2026-05-01",
            "Content-Type": "application/json",
        },
        json={"kausateId": kausate_id, "customerReference": customer_ref},
    )
    r.raise_for_status()
    return r.json()["orderId"]

async def fan_out(kausate_ids):
    async with httpx.AsyncClient(timeout=30) as client:
        # Submit in bounded batches to avoid 429 / connection storms
        sem = asyncio.Semaphore(20)
        async def one(kid):
            async with sem:
                return await submit_report(client, kid, f"sweep-{kid}")
        return await asyncio.gather(*[one(kid) for kid in kausate_ids])
```

The webhook receiver matches deliveries back to your sweep via the `customerReference` you set (or the `orderId` returned at submit time). Persist the `orderId → kausateId` mapping when you submit; correlate on receipt. See `../async-webhooks.md` for the full receiver pattern.

## B.6 — What Orbis has that Kausate doesn't

Be honest with yourself about which Orbis features your product actually uses. If any of these are load-bearing, you need a different / additional source:

- **M&A / deals.** Zephyr (BvD's M&A product) is the underlying dataset for Orbis M&A views. Kausate doesn't ingest deal databases or press releases.
- **Industry ratios + peer comparison.** Orbis computes financial ratios and benchmarks against a peer set with proprietary strength indicators. Kausate gives you the published financials when registries expose them; you derive ratios yourself or use a financial-data vendor.
- **Public-company filings (SEC 10-K equivalents).** Orbis harmonises EDGAR + global equivalents. Kausate doesn't.
- **Credit ratings.** Orbis surfaces credit and ESG scores plus the proprietary "MORE" credit model. Kausate doesn't.
- **Compliance flags (sanctions / PEPs / adverse media) via Grid.** Orbis surfaces these inside the Orbis record. Kausate doesn't — use Grid (kept), Refinitiv, Dow Jones, or ComplyAdvantage separately.
- **Pre-computed ownership graphs / ultimate-parent / subsidiary trees.** Orbis's "Ownership Explorer" entitlement is a separate paid line. Kausate's `POST /v2/companies/shareholder-graph` is live-pulled with `maxDepth` 1–7 — broadly comparable in *output*, but pulled fresh per call rather than served from a pre-computed graph.

Kausate is registry data, not a financial database. If you currently use Orbis as both, plan for a multi-vendor migration: Kausate for registry, something else for the analytics.

## B.7 — Migration footguns (Orbis-specific)

1. **"Orbis is an extract, not a live registry."** Caching Orbis responses for days is fine; caching Kausate `POST /v2/companies/report` responses for the same window invalidates the "live" guarantee that customers pay for. Kausate's same-day dedup is server-side — don't add a client-side cache that extends it. See `../status-and-dedup.md`.

2. **`Match` (≤50 results) vs `Companies` (≤4 000) — they served different use cases.** `Match` was the KYB-onboarding tool (strict, ranked, top-50). `Companies` was population sweeps. Kausate's `POST /v2/companies/search` is closer to `Match` — narrow, strict-ish, returns the top matches per call. For Orbis `Companies`-style population sweeps, use the bulk fan-out pattern in B.5.

3. **No saved-query model.** Orbis's `CreateQuery2` + `GetData2` pattern stored a strategy (WHERE / SELECT / TOP) server-side and re-executed it. Kausate has no equivalent — every call carries its full body. If you had query templates in Orbis, port them to client-side request builders that emit the equivalent Kausate request body each time.

4. **No bulk-ID hydration.** `SearchByBvDIds` doesn't have a one-to-one mapping. Fan out per-company.

5. **Sub-second latency expectations.** Orbis was a DB lookup — sub-second. Kausate is a live registry call — seconds to minutes. UIs designed for Orbis's response time will hang. Add async UX (spinner, "we'll notify you when the report is ready") for any user-facing flow.

6. **`429 Too Many Requests` semantics differ.** In Orbis REST, `429` is the canonical concurrency error — your account's simultaneous-call ceiling. In Kausate, `429` is rate-limit-based with retry-after-style backoff. Both call for the same client behaviour (back off + retry), but the **cause is different**: Kausate's ceiling is rate-based per minute, not concurrency-based per call.

---

# Part C — Maxsight → Kausate

Maxsight (formerly Passfort) is Moody's KYC/KYB workflow SaaS. It orchestrates third-party providers (including kompany and Orbis), adds a Profile state machine, review queues, multi-jurisdiction policy packs, and an analyst back-office UI. **It's not a registry data API** — it's the workflow layer on top.

> Maxsight DOES have a developer API (`developer.maxsight.com/api`, base URLs `api.{us,eu,ae}.maxsight.com/4.0/...`), webhooks (`help.maxsight.com/en/how-to-configure-webhooks.html`), and PEP/sanctions screening via API (`help.maxsight.com/en/run-a-peps-and-sanctions-screening-check-using-the-api`). It's not "UI-only." If your integration is API-driven, see C.2.

## C.1 — Two reader patterns

You probably fall into one of these:

1. **API-only consumer of Maxsight.** You consume Maxsight's REST API (`POST /4.0/profiles` etc.) and receive `CHECK_COMPLETED` webhooks. Migration is endpoint-level — replace Maxsight's data primitives with Kausate ones, replace the webhook receiver. See C.3.

2. **Workflow consumer.** Your team uses the Maxsight UI to run KYC cases — Profile creation, check execution, analyst review, decision sign-off. Migration is **product work**, not endpoint work — you replace the Maxsight UI with your own product, use Kausate for the registry-data layer, and keep PEP/sanctions/document-verification with whichever provider you're using under Maxsight (or directly with that provider after migration).

These two patterns share the same data primitives but have very different scope. C.3 focuses on (1); the workflow-consumer case is mostly about your own product roadmap.

## C.2 — Maxsight API surface (what to migrate FROM)

Base URLs (region-scoped):

```
https://api.us.maxsight.com/4.0/...   # US tenants
https://api.eu.maxsight.com/4.0/...   # EU tenants
https://api.ae.maxsight.com/4.0/...   # UAE tenants
```

The unit of work is a **Profile** (an individual or a company being screened). The pattern is:

```
POST /4.0/profiles
{
  "role": "INDIVIDUAL_CUSTOMER" | "COMPANY_CUSTOMER",
  "collected_data": { ... }
}
# → returns profile_id

POST /4.0/profiles/{profile_id}/events/{event_id}
# Runs / progresses a "check" against the profile.
```

Check types ("tasks") include — verbatim names where Google indexed the help-centre:

| Task | Triggering check |
|---|---|
| `INDIVIDUAL_ASSESS_MEDIA_AND_POLITICAL_AND_SANCTIONS_EXPOSURE` | PEPs + sanctions + adverse media on a person |
| `COMPANY_ASSESS_MEDIA_AND_SANCTIONS_EXPOSURE` | Company sanctions + adverse media |
| `COMPANY_WEBSITE_CONTENT` | Website content check; result `Pass` / `Refer` / `Error` |
| (Identity check, document verification, document fetch, device fraud, merchant fraud, visa check, company screening) | Other check families per `help.maxsight.com/en/check-types` |

**Webhook callbacks** are configured per-tenant in the UI (`Manage account > Webhook config > Webhooks enabled`):

- **Event type:** `event: "CHECK_COMPLETED"` in the payload
- **Top-level payload keys:** `secret` (shared-secret echo for validation), `id` (check ID), `data` envelope
- **Inside `data.check`:** `check_type` (e.g. `COMPANY_WEBSITE_CONTENT`), `variant.alias`, `result` (`Pass | Refer | Error` for the website-content variant)
- **Batching:** opt-in via `Enable batching`. When enabled, Maxsight delivers **the first 100 messages in the queue approximately every 0–10 seconds**.
- **Salesforce limitation:** "Webhooks cannot be sent directly to a Salesforce endpoint because Salesforce interprets the Authorization header as an attempt to authorize requests. Requests directly to a Salesforce endpoint will be rejected with a 401 Unauthorized Error." Workaround: an intermediary / proxy / webhook transformer.

**Demo tenant:** sign in with real credentials + `+demo` appended to the email local part (e.g. `morgan@forexo.com+demo`). UI shows a "Demo" badge. Paid checks are simulated; demo data is returned.

## C.3 — Endpoint mapping (Maxsight → Kausate)

What's on Maxsight that Kausate can replace:

| Maxsight check / data primitive | Kausate equivalent |
|---|---|
| Company screening — find a company by name + country | `POST /v2/companies/search` (async) or `/v2/companies/search/sync` |
| Company report (Maxsight orchestrates kompany / Orbis under the hood) | `POST /v2/companies/report` |
| UBO check | `POST /v2/companies/ubo` |
| Ownership / shareholder graph | `POST /v2/companies/shareholder-graph` (`maxDepth` 1–7) |
| Company document verification (when sourced from registry filings) | `POST /v2/companies/documents/list` → `POST /v2/companies/documents` |
| Ongoing change monitoring (Maxsight schedules re-runs of checks) | `POST /v2/monitors` with `sources` + `categoriesFilter` |
| Form prefill (Maxsight `collected_data` autofill) | `POST /v2/companies/prefill` (sync, < 200 ms) |

What stays with Maxsight (or another vendor):

| Maxsight check | Why it stays |
|---|---|
| PEP / sanctions / adverse media (`INDIVIDUAL_ASSESS_MEDIA_..._EXPOSURE`, `COMPANY_ASSESS_MEDIA_..._EXPOSURE`) | Not in Kausate. Keep on Maxsight (its bundled Grid / Refinitiv / Dow Jones integrations) or move to a dedicated PEP/sanctions vendor. |
| Identity check (selfie + ID document) | Not in Kausate. Keep on Maxsight or move to a dedicated identity vendor (Onfido, Veriff, Sumsub, Persona). |
| Document fetch (passport, driving licence; e.g. GBG IDscan GB variant) | Not in Kausate. |
| Device fraud detection / merchant fraud | Not in Kausate. |
| Visa check | Not in Kausate. |
| Profile state machine, review queues, multi-jurisdiction policy packs, analyst UI | These are the *workflow layer* itself. If you're a workflow consumer, you rebuild this in your own product; if you're an API-only consumer, you keep Maxsight for the orchestration. |

## C.4 — Webhook receiver migration

**Maxsight (before):**

```ts
// Maxsight POSTs CHECK_COMPLETED events; you verify via `secret` and route by check_type
app.post("/webhooks/maxsight", async (req, res) => {
  const payload = req.body;
  if (payload.secret !== process.env.MAXSIGHT_WEBHOOK_SECRET) {
    return res.status(401).end();
  }
  if (payload.event !== "CHECK_COMPLETED") {
    return res.status(200).end();
  }
  switch (payload.data.check.check_type) {
    case "COMPANY_WEBSITE_CONTENT": await routeWebsiteResult(payload); break;
    case "COMPANY_ASSESS_MEDIA_AND_SANCTIONS_EXPOSURE":
      await routeSanctionsResult(payload);
      break;
    // ...
  }
  res.status(200).end();
});
```

**Kausate (after) — registry-data slot replaced; route by `result.type`:**

```ts
// Kausate POSTs ExecutionResponse to a single org-wide subscription
app.post("/webhooks/kausate", async (req, res) => {
  // Auth via the customHeader you set at subscription time
  if (req.header("Authorization") !== `Bearer ${process.env.KAUSATE_WEBHOOK_SECRET}`) {
    return res.status(401).end();
  }
  const payload = req.body;

  // Idempotency check on orderId
  if (await alreadyProcessed(payload.orderId)) return res.status(200).end();

  // Handle every closed status, not just "completed"
  if (payload.status !== "completed") {
    return handleClosedNonSuccess(payload, res);  // see status-and-dedup.md
  }

  switch (payload.result?.type) {
    case "liveSearch":       await routeSearch(payload); break;
    case "companyReport":    await routeReport(payload); break;
    case "uboReport":        await routeUbo(payload); break;
    case "shareholderGraph": await routeGraph(payload); break;
    case "listDocuments":    await routeDocList(payload); break;
    case "document":         await routeDocument(payload); break;
  }
  res.status(200).end();
});
```

Key differences vs Maxsight's webhook:

- **No shared-secret echo.** Kausate authenticates webhooks via the `customHeaders` you set at subscription time (typically `Authorization: Bearer <your-shared-secret>`). Kausate does **not** HMAC-sign payloads.
- **One subscription per org, not per check type.** All order types your org places deliver to the same URL. Switch on `result.type`.
- **Acknowledge in 2xx within 30 s.** Same constraint as Maxsight, different timeout limit. Queue real work; respond fast.
- **Idempotency on `orderId`.** Kausate retries up to 50 times with exponential backoff. Make your handler safe to receive the same payload twice.
- **No batching.** Maxsight batches up to 100 messages every 0–10 s; Kausate delivers one webhook per order. If you relied on batching, that pattern doesn't carry over.

The full webhook receiver checklist is in `../async-webhooks.md`. Monitor change-events (separate channel, snake_case payload) are in `../monitors.md` — don't reuse one handler for both.

## C.5 — Maxsight integration providers — what you're actually migrating from

Under the hood, Maxsight orchestrates several providers. If you're moving registry-data lookups to Kausate, you're effectively re-pointing one slot in Maxsight's provider matrix:

| Maxsight integration | Underlying API | After migrating to Kausate |
|---|---|---|
| `Kompany KYC API` (`help.maxsight.com/en/configuring-kompany-kyc-api`) | kompany v2 | **Replace with Kausate** — see Part A |
| `Moody's Orbis` (`/configuring-moody-s-orbis`) | Orbis REST (Compliance Catalyst) | **Replace with Kausate** for registry-data fields — see Part B; keep Orbis for analytics if needed |
| `Moody's Grid` (`/configuring-moody-s-grid`) | Grid (PEP/sanctions/adverse-media) | **Keep** (or replace with Refinitiv / Dow Jones / ComplyAdvantage) |
| `Dow Jones Risk and Compliance` | Dow Jones | **Keep** |
| `Refinitiv World-Check One` | Refinitiv | **Keep** |
| `TransUnion TruValidate` (fraud) | TruValidate | **Keep** |
| `GBG IDscan` (document verification) | GBG | **Keep** |
| Salesforce / HRIS integrations | various | **Keep** (and proxy webhooks if Salesforce — see C.2 limitation) |

Practical migration step: in Maxsight's tenant settings, replace the kompany/Orbis integration with a "custom webhook" or wrap Kausate behind a thin proxy that exposes the kompany-shaped or Orbis-shaped API Maxsight expects. The Maxsight UI continues to drive the workflow; Kausate fills the data slot. After the proxy is stable, your product team can decide whether to keep Maxsight or rebuild the workflow in your own UI (the C.1 (2) path).

---

# Status mapping

kompany order strings + Orbis errors → Kausate's six statuses (see `../status-and-dedup.md`):

| kompany / Orbis / Maxsight | Kausate | Action |
|---|---|---|
| kompany `PROCESSING` | `running` | Wait for webhook or keep polling |
| kompany `COMPLETED` | `completed` | Use `result` |
| kompany `FAILED` | `failed` | Surface to user; retry with `bypassCache: true` if registry-side |
| kompany `CANCELLED` | `canceled` | Treat as not-found |
| (no kompany equivalent — would be `FAILED` in kompany) | `terminated` | Same as `failed` |
| (no kompany signal — kompany doesn't expose timeout) | `timedOut` | Retry — registries can be slow; next attempt may succeed |
| Orbis SOAP Fault (auth-failure on `Open`) | (n/a — auth-failures return 4xx, not an order status) | Fix the API key / version pin |
| Orbis REST `429` | (n/a — rate-limit, not an order status) | Back off + retry; per-org concurrency ceiling |
| Maxsight `CHECK_COMPLETED` with `data.check.result: Pass` | `completed` with `result.type` matching the check | Use `result` |
| Maxsight `CHECK_COMPLETED` with `data.check.result: Refer` | `completed` (Kausate has no "needs-human-review" state) | Your application surfaces the data; the review is your workflow's responsibility |
| Maxsight `CHECK_COMPLETED` with `data.check.result: Error` | `failed` | Surface to user; retry if appropriate |

**Don't write code that only checks `status === "completed"`** — handle all six. See `../status-and-dedup.md`.

---

# Before / after — three worked examples

## Example 1 — kompany v1 master-dataset GmbH lookup → Kausate report

**kompany (before, v1):**

```ts
// 1. index search to get the 32-char hex internal id
const search = await fetch(
  `https://api.kompany.com/api/v1/company/search/DE/${encodeURIComponent("HRB 31248")}`,
  { headers: { user_key: process.env.KOMPANY_KEY! } }
).then(r => r.json());

const kompanyId = search.data[0].id;
// kompanyId = "77A5AD48BBB33E5E3A397FFC920C3885"

// 2. fetch the "master" dataset (cached) — or "refresh" for live
const master = await fetch(
  `https://api.kompany.com/api/v1/company/${kompanyId}/master`,
  { headers: { user_key: process.env.KOMPANY_KEY! } }
).then(r => r.json());

// 3. if you want a live registry pull instead:
const refresh = await fetch(
  `https://api.kompany.com/api/v1/company/${kompanyId}/refresh`,
  { headers: { user_key: process.env.KOMPANY_KEY! } }
).then(r => r.json());
```

**Kausate (after, `2026-05-01`):**

```ts
// 1. translate native id → kausateId (cache it after first call)
const search = await kausate.POST("/v2/companies/search/sync", {
  body: { companyNumber: "HRB 31248", jurisdictionCode: "de" },
});
const kausateId = search.data.result.searchResults[0].kausateId;
// kausateId = "co_de_..."

// 2. place async report order — webhook delivers the result
const order = await kausate.POST("/v2/companies/report", {
  body: {
    kausateId,
    customerReference: "kyc-case-12345",
    // Equivalent of kompany "super": force a fresh registry call
    // bypassCache: true,
  },
  headers: { "X-Customer-Id": "end-customer-789" },
});
// order.data.orderId — persist this; webhook delivery will reference it.
// status = "running" initially; webhook arrives when complete.
```

Receiver-side: see Example 3 / `../async-webhooks.md`.

## Example 2 — Orbis SOAP `GetData2` → Kausate report

**Orbis (before):**

```xml
<!-- 1. Open a session -->
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <Open xmlns="http://bvdinfo.com/orbis/">
      <User>tenant-user</User>
      <Password>secret</Password>
    </Open>
  </soap:Body>
</soap:Envelope>
<!-- → session token returned in OpenResult -->

<!-- 2. CreateQuery2 — build a strategy with WHERE / SELECT / TOP -->
<CreateQuery2 xmlns="http://bvdinfo.com/orbis/">
  <SessionToken>...</SessionToken>
  <Query>
    <Where>BvDEPID = "DE12345678"</Where>
    <Select>
      <Field>CompanyName</Field>
      <Field>RegisteredAddress</Field>
      <Field>LegalRepresentatives</Field>
      <Field>ShareCapital</Field>
    </Select>
    <Top>1</Top>
  </Query>
</CreateQuery2>
<!-- → QueryId returned -->

<!-- 3. GetData2 — execute -->
<GetData2 xmlns="http://bvdinfo.com/orbis/">
  <SessionToken>...</SessionToken>
  <QueryId>...</QueryId>
</GetData2>
<!-- → DataResult with the requested fields -->

<!-- 4. Close -->
<Close xmlns="http://bvdinfo.com/orbis/">
  <SessionToken>...</SessionToken>
</Close>
```

**Kausate (after):**

```python
import httpx

API = "https://api.kausate.com"
HEADERS = {
    "X-API-Key": KAUSATE_API_KEY,
    "Kausate-Version": "2026-05-01",
    "Content-Type": "application/json",
}

# 1. Translate BvDEPID → kausateId (strip country prefix, lowercase, search)
search = httpx.post(
    f"{API}/v2/companies/search/sync",
    headers=HEADERS,
    json={"companyNumber": "12345678", "jurisdictionCode": "de"},
    timeout=15,
).json()
kausate_id = search["result"]["searchResults"][0]["kausateId"]

# 2. Place async report order — webhook delivers the full ExecutionResponse
order = httpx.post(
    f"{API}/v2/companies/report",
    headers={**HEADERS, "X-Customer-Id": "tenant-user"},
    json={
        "kausateId": kausate_id,
        "customerReference": "orbis-migration-001",
    },
    timeout=15,
).json()
order_id = order["orderId"]
# Webhook arrives at your registered URL with the company report.
# No session to close — Kausate is stateless.
```

The `Query`'s WHERE/SELECT/TOP model collapses: `WHERE BvDEPID = ...` becomes the `kausateId` body field; `SELECT` becomes "the canonical company report at version `2026-05-01`" (you receive all fields the endpoint returns; you select client-side); `TOP 1` is implicit (one company per order).

## Example 3 — Maxsight webhook receiver → Kausate webhook receiver

**Maxsight (before):**

```ts
// Express handler for Maxsight CHECK_COMPLETED
app.post("/webhooks/maxsight", async (req, res) => {
  const { secret, event, data, id } = req.body;

  // Verify via shared-secret echo
  if (secret !== process.env.MAXSIGHT_WEBHOOK_SECRET) return res.status(401).end();
  if (event !== "CHECK_COMPLETED") return res.status(200).end();

  const checkType = data.check.check_type;
  const variant = data.check.variant?.alias;
  const result = data.check.result;  // Pass | Refer | Error

  switch (checkType) {
    case "COMPANY_WEBSITE_CONTENT":
      await persistWebsiteResult(id, variant, result);
      break;
    case "COMPANY_ASSESS_MEDIA_AND_SANCTIONS_EXPOSURE":
      await persistSanctionsResult(id, data);
      break;
    // ...
  }

  res.status(200).end();
});
```

**Kausate (after):**

```ts
// Express handler for Kausate ExecutionResponse (order completions)
app.post("/webhooks/kausate", async (req, res) => {
  // 1. Auth via customHeader set at subscription time (no HMAC)
  if (req.header("Authorization") !== `Bearer ${process.env.KAUSATE_WEBHOOK_SECRET}`) {
    return res.status(401).end();
  }

  const payload = req.body;  // ExecutionResponse — see ../async-webhooks.md

  // 2. Idempotency on orderId (Kausate retries up to 50x)
  if (await alreadyProcessed(payload.orderId)) {
    return res.status(200).end();
  }

  // 3. Handle every closed status, not just "completed"
  if (payload.status !== "completed") {
    // Branch on running / failed / canceled / terminated / timedOut
    // See ../status-and-dedup.md
    await handleClosedNonSuccess(payload);
    return res.status(200).end();
  }

  // 4. Route by result.type (single subscription receives all order types)
  switch (payload.result?.type) {
    case "liveSearch":
      await persistSearch(payload);
      break;
    case "companyReport":
      // This is what replaces Maxsight's kompany-/Orbis-backed company report
      await persistCompanyReport(payload);
      break;
    case "uboReport":
      await persistUbo(payload);
      break;
    case "shareholderGraph":
      await persistGraph(payload);
      break;
    case "listDocuments":
      await persistDocList(payload);
      break;
    case "document":
      // result.downloadLink is a pre-signed S3 URL — fetch the bytes
      // before result.expiresAt and persist them on your side
      await fetchAndPersistDocument(payload);
      break;
  }

  // 5. Acknowledge fast (Kausate's HTTP timeout is 30 s)
  res.status(200).end();
});
```

PEP/sanctions/adverse-media events stay on Maxsight's webhook receiver — Kausate doesn't carry that data. You'll end up with two webhook receivers (Kausate + whatever you keep for screening), each handling its own slice.

---

# Reference index

- **Auth, version pinning, generated client** — `../auth-versioning.md`
- **Endpoint inventory + native-id → `kausateId` translation** — `../endpoints.md`
- **Async lifecycle + webhook receiver pattern** — `../async-webhooks.md`
- **`customerReference` vs `customerId` semantics** — `../customer-correlation.md`
- **All six order statuses + same-day dedup + `bypassCache`** — `../status-and-dedup.md`
- **Document retrieval (two-step flow, pre-signed URLs)** — `../documents.md`
- **Change monitoring (cron sources, feed sources, event taxonomy)** — `../monitors.md`
- **Per-jurisdiction native ID formats** — `../identifiers.md`
- **Live OpenAPI spec** (auth required) — `https://api.kausate.com/openapi.json`
- **Human-readable docs** — `https://docs.kausate.com`

If anything here conflicts with `openapi.json`, the spec wins.
