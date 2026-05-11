# Migrating from Kyckr to Kausate

The technical migration path for code running against Kyckr. The job: map existing Kyckr code onto Kausate equivalents and surface the differences that bite during the switch.

---

## 1. Mental-model translation

Architecturally similar: a real-time REST surface over hundreds of business registries, with synchronous search/profile and async "buy a thing from the registry" flows for documents and ownership trees. What's moving is **payload shape, status semantics, and how completions get to you**.

**First — which Kyckr are you on?** Kyckr ships two parallel production APIs:

| Surface | Host | Path style |
|---|---|---|
| Companies **V1** ("Core" / legacy) | `https://rest.kyckr.com` | `/core/<resource>/<verb>/{ISO}/{Code}`, `/lite/profile/...` |
| Companies **V2** ("modern") | `https://api.kyckr.com` | `/v2/companies`, `/v2/companies/{kyckrId}/...`, `/v2/orders` |

Look at your base URL: `rest.kyckr.com` (or the older `kyckr.azure-api.net/core/...`) → V1; `api.kyckr.com/v2/...` → V2. V1 is feature-complete (it's the only surface with UBO Verify); V2 is the cleaner shape. V1 migrators have more to port; V2 migrators mostly need to flip auth, swap identifiers, and switch to webhooks.

Three deltas dominate regardless of version:

1. **Webhooks first.** Kyckr is poll-only — no webhook in V1 or V2. Completions arrive by polling `order-status` (V1) or `GET /v2/orders/{id}` (V2). Kausate returns `{orderId, status: "running"}` immediately and the result lands on your registered webhook. Polling stays as a fallback (`GET /v2/companies/{family}/{orderId}`).
2. **Stable `kausateId`.** V1 keys on `(CountryISO, codeField[, RegAuth])`; V2 keys on the volatile `kyckrId` (`{ISO}|{base64}` — Kyckr says **don't persist it**). Kausate gives you one durable `kausateId` per company. DE/CA `RegAuth` round-trip and V2 base64 churn both disappear.
3. **Date-pinned API version.** Every request sends `Kausate-Version: 2026-05-01`. No more V1-vs-V2 host fork — breaking changes are on your schedule.

---

## 2. Quick endpoint-mapping table

| Capability | Kyckr V1 | Kyckr V2 | Kausate |
|---|---|---|---|
| Search by name / number | `GET /core/company/search/{ISO}/{NameOrNumber}` | `GET /v2/companies?name=&isocode=&companyNumber=` | `POST /v2/companies/search` (async) or `/search/sync` |
| Global search (no jurisdiction) | `GET /lite/search/{name}` *(path under-documented)* | `GET /v2/companies?name=` *(no isocode)* | `POST /v2/companies/search` with no `jurisdictionCode` |
| Search by person | — (UBO output only) | — | `GET /v2/companies/search/person` |
| Autocomplete / typeahead | — | — | `GET /v2/companies/search/autocomplete` |
| Lite profile | `GET /lite/profile/{ISO}/{code}` | `GET /v2/companies/{kyckrId}/lite` | (no exact equivalent — closest: `POST /v2/companies/prefill`) |
| Enhanced / full profile | `GET /core/company/profile/{ISO}/{code}` | `GET /v2/companies/{kyckrId}/enhanced` | `POST /v2/companies/report` |
| Financials | (folded into Enhanced) | (folded into Enhanced) | `POST /v2/companies/finance` |
| List documents | `GET /core/filing/search/{ISO}/{code}` | `GET /v2/companies/{kyckrId}/documents` | `POST /v2/companies/documents/list` |
| Order document | `POST /core/filing/order` | `POST /v2/orders` | `POST /v2/companies/documents` |
| Poll document status | `GET /core/filing/order-status/{days}?orderRef=` | `GET /v2/orders/{orderId}` | webhook OR `GET /v2/companies/documents/{orderId}` |
| Download document | `urlField` (returned with status) | `links.download` on order | `result.downloadLink` (pre-signed, `expiresAt`) |
| Single-layer UBO | (no direct — see below) | — | `POST /v2/companies/ubo` |
| Multi-layer UBO / shareholder graph | `POST /core/uboverify/create/{ISO}/{code}` + `GET /core/uboverify/list/{orderId}` | — *(not in V2 nav as of 2026-05)* | `POST /v2/companies/shareholder-graph` |
| Change monitoring | Company Watch (sales-led, no public API) | (same — no public API) | `POST /v2/monitors` (self-serve) |
| Webhooks for completions | — (poll-only) | — (poll-only) | `POST /v2/webhooks` |
| Sanctions / PEP | not in public API | not in public API | not in Kausate API |
| Jurisdiction capability matrix | static.kyckr.com PDF / sales | static.kyckr.com PDF / sales | `GET /v2/platform/jurisdictions` |

Per-feature detail and parameter mappings are below.

---

## 3. Auth diff

```diff
- Authorization: {{apiKey}}              # Kyckr — raw key, NO Bearer prefix (both V1 and V2)
+ X-API-Key: $KAUSATE_API_KEY
+ Kausate-Version: 2026-05-01            # mandatory in production
```

- Kyckr puts the raw API key in `Authorization` with no scheme prefix. (One V2 lite-profile doc renders `Authorization: Bearer {{apiKey}}` — a doc inconsistency; the bare form is canonical.) On Kausate, stop writing `Authorization:` entirely — use `X-API-Key`.
- No Bearer / OAuth / JWT path on Kausate's public API. `X-API-Key` is the only auth.
- The Kyckr host fork (`rest.kyckr.com`, `api.kyckr.com`, `kyckr.azure-api.net`) collapses to one Kausate host: `https://api.kausate.com`. Versioning moves into the header.
- `Kausate-Version` is mandatory in production — without it you ride the org default. See `../auth-versioning.md`.

---

## 4. Identifier translation — one-time, then cached

Most code touches Kyckr-isms here; budget the most porting time.

**Kyckr V1** identifies a company by:
- `codeField` — Kyckr's opaque id, returned from V1 search.
- `{CountryISO}` — uppercase ISO 3166-1 alpha-2, sometimes with a sub-territory suffix (`US-TX`, `CA-ON`, `CA-ALL`).
- `RegAuth` — required for **DE** and **CA** only. When search returns `registrationAuthorityCodeField`, echo it as `RegAuth` on every subsequent profile / filing / UBO call.

**Kyckr V2** identifies a company by:
- `kyckrId` — `{jurisdictionISO}|{base64(registryRef)}`, e.g. `GB|MTE2NTUyOTA`. V2 workflow guide: *"This format may change and should not be persisted."* Re-derive via search.

**Kausate** identifies a company by:
- `kausateId = co_{jurisdiction}_{opaque}` (e.g. `co_de_7KGHtucR88u2omSx3KhaoH`). Stable, persist it. **No `RegAuth` analogue** — search returns one disambiguated `kausateId`. Jurisdiction codes are **lowercase** (`gb`, `de`, `us-tx`, `ca-on`).

**The translation flow:**

```bash
# Kyckr V1 (before)
curl 'https://rest.kyckr.com/core/company/search/GB/11655290' \
  -H 'Authorization: {{apiKey}}'
# → companiesField[0].codeField  (keyed everything on this + "GB")

# Kyckr V2 (before)
curl 'https://api.kyckr.com/v2/companies?companyNumber=11655290&isocode=GB' \
  -H 'Authorization: {{apiKey}}'
# → data[0].id == "GB|MTE2NTUyOTA"  (don't persist — re-derive)

# Kausate — one shot, persist the result
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyNumber":"11655290","jurisdictionCode":"gb"}'
# → result.searchResults[0].kausateId == "co_gb_..."
```

Translate at the boundary: run search once, persist `(legacy_id → kausateId)`, use `kausateId` everywhere else.

---

## 5. Feature-by-feature

### 5.1 Search

**Kyckr V1** — `GET /core/company/search/{ISO}/{Name-or-Number}`. The same path resolves a name or a registry number; Kyckr disambiguates server-side. Query params: `RegAuth` (required where the registry has sub-authorities), `OrderRef` (free-text customer ref echoed in response and billing). No `pageSize` / cursor — `responseCodeField=109 TOO_MANY_MATCHES` is the only overflow signal.

**Kyckr V2** — `GET /v2/companies?name=...&isocode=...&companyNumber=...`. Same endpoint handles jurisdiction-scoped search and global search (omit `isocode`). Real pagination: `pageNumber` (1-based), `pageSize` (default 50), `totalCount`, `totalPages`.

**Kausate** — `POST /v2/companies/search` (async) or `POST /v2/companies/search/sync` (sync).

Body fields:

| Kyckr V1 / V2 | Kausate | Notes |
|---|---|---|
| `{ISO}` path segment / `isocode` query | `jurisdictionCode` (body) | **Lowercase** in Kausate (`gb` not `GB`). For US/CA sub-territories: `us-tx`, `ca-on` (lowercase, hyphenated). |
| `{Name-or-Number}` path segment | `companyName` **or** `companyNumber` (separate body fields) | Kausate splits them — no server-side disambiguation between the two. |
| (none) | `advancedQuery` | New: structured query for jurisdiction-specific identifiers. Use when you have a native registry number (KVK, HRB, SIREN) — avoids fuzzy matching. |
| `RegAuth` | (gone) | Disambiguation happens before `kausateId` is minted. |
| `OrderRef` | `customerReference` | Body field, not query. Per-request, ≤150 URL-safe chars. See `../customer-correlation.md`. |

Response field mapping:

| Kyckr V1 `companiesField[]` / V2 `data[]` | Kausate `result.searchResults[]` |
|---|---|
| `codeField` (V1) / `id` (V2, the `kyckrId`) | `kausateId` |
| `companyIDField` (V1) / `companyNumber` (V2) | `companyNumber` |
| `nameField` (V1) / `companyName` (V2) | `companyName` |
| `legalFormField` | `legalForm` |
| `legalStatusField` ∈ `ACTIVE` / `INACTIVE` / `DISTRESSED` | `status` (string) — see `../status-and-dedup.md` for the order-level status enum |
| `registrationAuthorityField` / `registrationAuthorityCodeField` | (collapsed into `kausateId`) |
| `addressesField[]` | `addresses[]` |

```ts
// Before — Kyckr V1
const res = await fetch(
  `https://rest.kyckr.com/core/company/search/GB/Kausate`,
  { headers: { Authorization: process.env.KYCKR_KEY! } },
).then(r => r.json());
const code = res.companiesField[0].codeField;          // persist (code, "GB")

// After — Kausate (sync variant — see async-webhooks.md for async)
const res = await kausate.POST("/v2/companies/search/sync", {
  body: { companyName: "Kausate", jurisdictionCode: "gb" },
});
const kausateId = res.data.result.searchResults[0].kausateId;  // persist
```

**Gotchas**

- `responseCodeField=109 TOO_MANY_MATCHES` (V1) → Kausate just returns a truncated list. Narrow with `companyNumber` or `advancedQuery`; no equivalent error code.
- V1 has no search pagination; V2 introduced it; Kausate's async search returns a single page — refine the query instead of paging.
- Kyckr's global search (V2 no `isocode`; V1 `/lite/search/...`) is **cache-backed and possibly stale** — they recommend re-confirming with a jurisdiction call. Kausate's live search hits the registry; no re-confirm needed.

### 5.2 Lite profile

**Kyckr V1** — `GET /lite/profile/{ISO}/{codeField}[?RegAuth=...]`. **V2** — `GET /v2/companies/{kyckrId}/lite`. Returns: name, number, registered address, registration/foundation date, registration/legal status, legal form, company activity, registration authority. Sufficient for "does this company exist and is it ACTIVE."

**Kausate has no exact equivalent.** The closest fast endpoint is:

```
POST /v2/companies/prefill
```

`prefill` is always sync, hits Kausate's company index first and falls back to a live registry call when missing, typically returns in <200 ms. It's optimized for **form-prefill UX** (user types a number → instantly populate the form), not for "render a lite profile."

Mapping:

| Kyckr lite (V1/V2) | Kausate `prefill` |
|---|---|
| `nameField` / `companyName` | `companyName` |
| `companyIDField` / `companyNumber` | `companyNumber` |
| Registered address | `registeredAddress` |
| Foundation / registration date | `registrationDate` |
| `legalStatusField` (ACTIVE/INACTIVE/DISTRESSED) | `status` (normalized similarly) |
| `legalFormField` | `legalForm` |
| Activity code + description | `activity` |

If your use case is *"verify the company exists and is active before doing anything else"*, `prefill` does it cheaper and faster than Kyckr's lite. If you need the full registry-backed payload, go straight to `report` (§5.3) — Kausate doesn't separately bill a "lite" tier.

### 5.3 Enhanced / full profile

**Kyckr V1** — `GET /core/company/profile/{ISO}/{codeField}[?RegAuth=...]`. **V2** — `GET /v2/companies/{kyckrId}/enhanced`. Adds, on top of lite (where the registry publishes them): share capital structure, company officials / directors, shareholders, **registry-filed** UBOs (CZ, LU, IE, NL, DE filings). Shape is **jurisdiction-shaped, not fully normalized** — only `legalStatusField` is normalized.

**Kausate** — `POST /v2/companies/report` (async; webhook delivery). Sync variant at `/v2/companies/report/sync` exists for prototypes.

Body mapping:

| Kyckr param | Kausate body | Notes |
|---|---|---|
| `{ISO}` + `{codeField}` (V1) / `{kyckrId}` (V2) | `kausateId` | One field, no jurisdiction needed alongside. |
| `RegAuth` (DE/CA only) | (gone) | |
| `OrderRef` | `customerReference` | |

Response: Kyckr returns the JSON synchronously. Kausate returns `{orderId, status: "running"}` immediately and POSTs an `ExecutionResponse` to your webhook with `result.type = "companyReport"`. See `../async-webhooks.md`.

```ts
// Before — Kyckr V1 enhanced (blocks until registry replies; can be slow)
const r = await fetch(
  `https://rest.kyckr.com/core/company/profile/GB/${codeField}`,
  { headers: { Authorization: process.env.KYCKR_KEY! } },
).then(r => r.json());
const officials = r.officialsField;
const shareholders = r.shareholdersField;

// After — Kausate (async + webhook)
const order = await kausate.POST("/v2/companies/report", {
  body: { kausateId, customerReference: "kyc-case-12345" },
  headers: { "X-Customer-Id": "tenant-789" },
});
// order.data.orderId — match the webhook delivery against this
// In your webhook handler:
//   if (payload.result?.type === "companyReport") {
//     const { officials, shareholders, ...rest } = payload.result;
//     ...
//   }
```

**Gotchas**

- Kyckr enhanced is *synchronous-but-slow* — if you've hit edge timeouts (Vercel: 30s, Cloudflare: ~30s, ELB: 60s) on Kyckr, you know why. Kausate moves this off the request path.
- Officials, share classes, role labels are **registry-shaped** on both. Kyckr: *"names, legal forms, role labels are passed through from the source register"* — same on Kausate. Only `legalStatus` is normalized.
- Financials are folded into Kyckr's enhanced response. On Kausate, financials are a separate endpoint: `POST /v2/companies/finance`. If you parse `annualAccountsField` off Kyckr enhanced today, that branch needs to move into a `finance` handler.
- Kyckr V2 enhanced includes a `links.document` to a renderable PDF of the profile. Kausate doesn't render the profile as a PDF — render your own from `result.companyReport`, or order a registered extract via `documents` (§5.4).

### 5.4 Document list, order, status, download

This is one logical capability with three calls on Kyckr V1 and two on V2; on Kausate it's the same two async orders for everyone (list → fetch).

#### List documents

**Kyckr V1** — `GET /core/filing/search/{ISO}/{codeField}[?RegAuth=...&ContinuationKey=...]`. **V2** — `GET /v2/companies/{kyckrId}/documents`.

**Kausate** — `POST /v2/companies/documents/list`. Async; webhook delivers `result.type = "listDocuments"` with an array of `kausateDocumentId`. See `../documents.md`.

Response field mapping for the per-document entries:

| Kyckr V1 field | Kyckr V2 field | Kausate `result.documents[].` |
|---|---|---|
| `idField` / `productCodeField` | `id` | `kausateDocumentId` |
| `productTitleField` | `name` | `title` |
| `productFormatField` (`application/pdf`) | `documentFormat[0]` | `fileType` (`pdf`, etc.) |
| `tierValueField` (cost in Kyckr Credits) | `cost.value` | — *(billing is decoupled — see analytics)* |
| `priceField` / `vatChargeField` / `currencyField` (registry currency) | — | — |
| `deliveryTimeMinutesField` | `deliveryTimeMinutes` | — *(implicit in async lifecycle)* |
| `typeField` (registry bucket) | — | `documentType` (Kausate-normalized enum: `annualAccounts`, `currentExtract`, `articlesOfAssociation`, `shareholderList`, `chronologicalExtract`, `officialFilings`, `registrationDetails`, `beneficialOwnersDetails`, `annualReturn`) |
| `displayDateField` | — | `publicationDate` |
| `continuationKeyField` (top-level, paginates) | (V2 pagination on outer list) | — *(Kausate returns the full list)* |

**Important:** Kyckr's `productCodeField` is **registry-specific** — there's no global catalogue. GB maps to Companies House filing categories; DE to Handelsregister product slots; FR to Infogreffe/INPI extracts; NL to KvK uittreksel + jaarrekening; etc. **Don't try to literal-map `productCodeField` values to anything in Kausate.** Kausate gives you `documentType` — a small, normalized enum — and `title` — the registry-native human label. Route on `documentType`; show `title` to the user.

#### Order, status, download

**Kyckr V1** — three calls:

```bash
# 1. Order
curl -X POST 'https://rest.kyckr.com/core/filing/order' \
  -H 'Authorization: {{apiKey}}' -H 'Content-Type: application/json' \
  -d '{ "countryIso": "GB", "productKey": "...", "companyName": "...", "orderRef": "case-1" }'

# 2. Poll status (recommended ~once per minute; ~90% deliver in <15 min)
curl 'https://rest.kyckr.com/core/filing/order-status/30?orderRef=case-1' \
  -H 'Authorization: {{apiKey}}'

# 3. Fetch the bytes from urlField (data.kyckr.com/integration/{id})
```

Kyckr V1 status payload:

```json
{ "statusField": 3, "urlField": "https://data.kyckr.com/integration/1603863699",
  "orderReferenceField": "case-1", "productOrderIdField": 2573359 }
```

`statusField` enum: `1` pending, `2`/`3`/`21` ready, `5` failed, `9` cancelled. Polling is **once-a-minute** recommended cadence. The signed-URL TTL on `urlField` is not documented — confirm with Kyckr support whether you can persist the link or must download immediately.

**Kyckr V2** — two calls: `POST /v2/orders` body `{kyckrId, productId}` → status string-typed (`Pending`/`Complete`/`Failed`) at `GET /v2/orders/{orderId}` → download via `/orders/{id}/download`.

**Kausate** — one call to place, then webhook (or poll):

```bash
curl -X POST https://api.kausate.com/v2/companies/documents \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId": "co_gb_...",
    "kausateDocumentId": "doc_...",
    "customerReference": "docs-fetch-1"
  }'
# → { "orderId": "ord_...", "status": "running" }
# Webhook delivers result.type = "document" with downloadLink + expiresAt
```

Parameter mapping:

| Kyckr V1 order body | Kyckr V2 order body | Kausate body |
|---|---|---|
| `countryIso` | (in `kyckrId`) | (in `kausateId`) |
| `productKey` / `productCodeField` | `productId` | `kausateDocumentId` (from list response) |
| `companyName` | (server-derived) | (server-derived from `kausateId`) |
| `orderRef` | `?customerReference=` query | `customerReference` body field |

Result field mapping:

| Kyckr V1 status field | Kyckr V2 | Kausate webhook `result` |
|---|---|---|
| `statusField` (1/2/3/5/9/21) | `status` (`Pending`/`Complete`/`Failed`) | top-level `status` (string, see §6) |
| `urlField` | `links.download` | `result.downloadLink` (pre-signed) |
| (none) | (none) | `result.expiresAt` (signed-URL TTL) |
| `productOrderIdField` | `orderId` | `orderId` |
| `orderReferenceField` | `customerReference` | `customerReference` |

**Gotchas**

- **Polling stops being your default.** The whole `/order-status/{days}?orderRef=` muscle memory goes away — webhook instead. Polling still exists at `GET /v2/companies/documents/{orderId}` for orders placed before you subscribed (no backfill).
- **`downloadLink` expires** (Kausate is explicit via `expiresAt`; Kyckr is silent on `urlField` TTL). Persist the bytes; **never persist the link itself**.
- **No `Authorization` header on the download fetch.** Kausate's `downloadLink` is signed — fetch it plain. Don't forward the API key.
- **No bulk-order endpoint on either side.** Client-side fan-out + reconciliation; on Kausate via webhook, on Kyckr via `?orderRef=` poll.

### 5.5 UBO Verify → shareholder graph

UBO Verify lives only under Kyckr V1 — no V2 equivalent in the published navigation as of 2026-05. If you're already on Kyckr V2 you've been mixing surfaces for this capability.

#### Create

**Kyckr** — `POST /core/uboverify/create/{ISO}/{companyCode}` with query params:

| Kyckr query param | Required | Meaning |
|---|---|---|
| `regAuth` | DE/CA only | Echo from search response |
| `maxCreditCost` | **yes** | Integer > 0. Hard cap on credits Kyckr will spend unwrapping. Stops when reached. |
| `uboThreshold` | optional | Ownership % to qualify as UBO (1–100, default `25`) |
| `maxLayers` | optional | Max depth of unwrap. Minimum `3`. |
| `continuationKeys` | optional | UUID(s) used to resolve a previous ambiguous match — see continuation flow below |

**Kausate** — `POST /v2/companies/shareholder-graph` (multi-layer) or `POST /v2/companies/ubo` (registry-filed UBOs only, single layer):

| Kyckr param | Kausate body | Notes |
|---|---|---|
| `{ISO}/{companyCode}` | `kausateId` | |
| `regAuth` | (gone) | |
| `maxLayers` | `maxDepth` | Range 1–7 (Kyckr's was min 3) |
| `uboThreshold` | (no param) | Apply threshold filtering on the returned graph in your own code |
| `maxCreditCost` | (no equivalent) | Kausate doesn't expose a hard credit cap — budget by `maxDepth`, `bypassCache`, and `customerId` cost tracking via `/v2/analytics/breakdowns?groupBy=customerId` |
| `continuationKeys` | (gone) | Disambiguation happens at search time — see below |
| — | `enriched` (optional) | Adds governance enrichment |
| — | `retrievalTimeout` (optional, minutes) | Per-order time budget |
| `OrderRef` (none on UBO) | `customerReference` | |

```bash
# Before — Kyckr
curl 'https://rest.kyckr.com/core/uboverify/create/GB/09657876?uboThreshold=25&maxCreditCost=50&maxLayers=5' \
  -H 'Authorization: {{apiKey}}'
# → { "ownershipTreeField": { "statusField": "SUCCESS", "orderIdField": "2614793" }, "responseCodeField": "201" }

# After — Kausate
curl -X POST https://api.kausate.com/v2/companies/shareholder-graph \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{ "kausateId": "co_gb_...", "maxDepth": 5, "enriched": true,
        "customerReference": "case-1" }'
# → { "orderId": "ord_...", "status": "running" }
# Webhook delivers result.type = "shareholderGraph"
```

#### Poll / list

**Kyckr** — `GET /core/uboverify/list/{orderId}`. Poll until `statusField != IN_PROGRESS`. Kyckr's cadence guidance is "once a minute" (same as filings); complex trees can take several minutes.

**Kausate** — webhook delivers the completed graph automatically. Polling fallback at `GET /v2/companies/shareholder-graph/{orderId}`, same `ExecutionResponse` shape.

#### Status enum mapping (UBO tree)

Kyckr's `statusField` for the UBO tree is richer than the per-order status; here's the translation:

| Kyckr `statusField` | Kausate top-level `status` | Where the detail goes |
|---|---|---|
| `IN_PROGRESS` | `running` | (still running) |
| `SUCCESS` | `completed` | `result.shareholderGraph.*` |
| `COMPANY_NOT_FOUND` | `failed` | `error` with a precise message; if the issue was ambiguity at the root, you'd have caught it at search time on Kausate |
| `NO_SHAREHOLDERS` | `completed` | `result.shareholderGraph.nodes = [root]`, `edges = []` |
| `REGISTRY_NOT_AVAILABLE` | `failed` (or `timedOut` if it stalled) | `error.message` |
| `INSUFFICIENT_CREDITS` | `failed` | Kausate exposes this as a 402-class billing error — top up the org |
| `INSUFFICIENT_MAX_CREDIT_COST` | (no analogue) | Kausate has no `maxCreditCost`; this category disappears |
| `INSUFFICIENT_MAX_LAYERS` | `completed` | Graph is returned as far as `maxDepth` walked; check `result.shareholderGraph.truncated` if present |
| `ORDER_NOT_FOUND` | (404 at the API) | |
| `SERVICE_UNAVAILABLE` | `failed` | |

#### Continuation flow disappears

Kyckr's continuation flow handles **ambiguous matches mid-walk**: when an intermediate shareholder company can't be uniquely identified, the LIST response halts with `COMPANY_NOT_FOUND`, populates `candidatesField` with each plausible match (each with its own `continuationKeyField` UUID), and you re-POST `create` with `?continuationKeys=<uuid1>,<uuid2>` to keep walking. Kyckr caches enhanced profiles for **24 hours** so the second walk doesn't double-pay.

**On Kausate this entire flow goes away.** Kausate disambiguates at *search* time — every node in the graph is already a `kausateId`, so there is no ambiguity to resolve mid-walk. There is no `continuationKeys` analogue, no `reasonForNonContinuationField`, no `candidatesField`. If a sub-entity genuinely couldn't be matched in the registry, it appears in the graph as a node with the most-confident match Kausate could make (or `null` for an unmatched person).

#### Tree-shape mapping

Kyckr's `ownershipTreeField` carries `nodesField[]`, `edgesField[]`, `ultimateBeneficialOwnersField[]`. Kausate's `result.shareholderGraph` carries the same logical structure:

| Kyckr UBO Verify | Kausate `shareholderGraph` |
|---|---|
| `nodesField[].nodeIdField` | `nodes[].nodeId` |
| `nodesField[].typeField` ∈ `PERSON`/`COMPANY`/`OTHER` | `nodes[].type` |
| `nodesField[].nameField` | `nodes[].name` |
| `nodesField[].companyCodeField` | (folded into `nodes[].kausateId` for COMPANY nodes) |
| `nodesField[].jurisdictionField` | (folded into `kausateId`) |
| `edgesField[].typeField = SHAREHOLDER` | `edges[].relationshipType` |
| `edgesField[].percentageField` | `edges[].percentage` |
| `edgesField[].isCircularField` | `edges[].isCircular` |
| `edgesField[].shareholdingsField[].isJointlyHeldField` (AU/NZ only) | `edges[].jointShareholding.isJoint` |
| `edgesField[].shareholdingsField[].jointHoldingGroupIdField` (UUID) | `edges[].jointShareholding.groupId` |
| `ultimateBeneficialOwnersField[].rollupPercentageField` | `result.ubos[].rolledUpPercentage` |

Kyckr defines a **12-value RFNC** enum on `reasonForNonContinuationField.typeField`: `COMPANY_NOT_FOUND`, `INSUFFICIENT_MAX_CREDIT_COST`, `REGISTRY_NOT_AVAILABLE`, `NO_SHAREHOLDERS`, `INSUFFICIENT_CREDITS`, `DESERIALIZATION_ERROR_ENHANCED_PROFILE`, `SERVER_ERROR`, `UNSUPPORTED_JURISDICTION`, `INSUFFICIENT_MAX_LAYERS`, `SHAREHOLDING_TOO_LOW`, `INVALID_REGAUTH`, `POSSIBLE_PLC`. Kausate doesn't surface a parallel enum — most of these don't apply on Kausate (`INVALID_REGAUTH`, `INSUFFICIENT_MAX_CREDIT_COST`, `DESERIALIZATION_ERROR_ENHANCED_PROFILE`, `POSSIBLE_PLC`) or collapse into per-node `walkStoppedReason` strings. If your Kyckr code routes on RFNC, drop most branches and rewrite the rest against simpler shapes.

**Joint shareholdings (AU/NZ).** Kyckr carries jointly-held shares on `shareholdingsField[]` with `isJointlyHeldField` + `jointHoldingGroupIdField`. Kausate's `edges[].jointShareholding.{isJoint, groupId}` carries the same idea — rename the fields.

**No UBO PDF report on either side.** Kyckr's portal renders a PDF; Kyckr V1's API doesn't expose one. Kausate is the same — `shareholder-graph` returns JSON; render your own PDF from `result` if you need one.

### 5.6 Company Watch / ongoing monitoring

Kyckr's **Company Watch** / **Ongoing Monitoring** product exists but is **sales-led, not self-serve**: no `/monitoring` or `/watch` endpoint on V1 or V2 developer.kyckr.com. Marketing pages describe four event classes — *new/removed directors*, *new/removed shareholders*, *changes to capital structure*, *changes to registration details* — across two tiers (Lite / Enhanced). Delivery channel is bespoke per customer (push / poll / email), no published pricing, no documented schema.

**Migrating means you finally get a self-serve, public-API change-monitoring product.** Map the Kyckr event classes to Kausate's `category` taxonomy:

| Kyckr Company Watch event class | Kausate `category` | Example `event_code`s |
|---|---|---|
| New / removed directors | `legalRepresentatives` | `DIRECTOR_ADDED`, `AUTHORIZED_SIGNATORY_CHANGED` |
| New / removed shareholders | `ownership` | `SHAREHOLDER_ADDED`, `UBO_PERCENTAGE_CHANGED` |
| Changes to capital structure | `financial` | `SHARE_CAPITAL_CHANGED`, `FINANCIAL_FILING_PUBLISHED` |
| Changes to registration details | `other` (and `address` for address-only) | `NAME_CHANGED`, `LEGAL_FORM_CHANGED`, `REGISTERED_ADDRESS_CHANGED` |
| (Kyckr also delivers a list of new filings within the window) | — | New filings are not a `monitor.change_detected` event; subscribe to `financial` (for accounts) and re-`list documents` to discover new ones |
| (not in Kyckr Company Watch) | `status` | `INSOLVENCY_OPENED`, `INSOLVENCY_DECISION`, `STATUS_CHANGED` — Kausate also ships insolvency-feed monitoring (DE Insolvenzbekanntmachungen today, more coming) |
| (not in Kyckr Company Watch) | `disappeared` | `COMPANY_DISSOLVED`, `COMPANY_STRUCK_OFF` — flips `is_active=false` automatically |

Create a monitor:

```bash
curl -X POST https://api.kausate.com/v2/monitors \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "kausateId": "co_de_...",
    "sources": ["company_report", "shareholder_graph",
                "de_insolvenzbekanntmachungen.feed"],
    "scheduleCron": "0 9 * * *",
    "webhookUrl": "https://your-app.example.com/webhooks/kausate-monitors",
    "categoriesFilter": ["legalRepresentatives", "ownership", "financial",
                         "status", "disappeared"]
  }'
```

Full reference: `../monitors.md`. **Don't reuse your async-completion webhook handler for monitor events** — different payload shape (snake_case, `event_code` / `category` discriminator) and different per-monitor URL.

### 5.7 Webhooks — the production-pattern flip

To be explicit: **Kyckr has no developer-API webhook**, V1 or V2. Order completion, UBO Verify completion, and any non-Company-Watch event must be polled at the recommended **once-a-minute** cadence. No HMAC, no subscription endpoint, no event types. Company Watch *may* deliver via push under a bespoke contract — but that's sales-provisioned.

Migrating to Kausate:

1. Subscribe **once** at onboarding via `POST /v2/webhooks` (see `../async-webhooks.md`). One subscription receives completions for every async endpoint; switch on `result.type`.
2. Send `customerReference` on every order so deliveries route back to your originating record.
3. Keep a reconciliation poll (`GET /v2/companies/{family}/{orderId}`) as a backstop for stragglers and orders placed before a subscription existed (no backfill on Kausate either).

The win is **latency**: webhooks fire on completion (seconds, not the next polling tick) and you stop carrying a job scheduler purely for status checks.

### 5.8 Sanctions / PEP

**Neither product has a public sanctions or PEP API.** Kyckr's marketing describes sanctions/PEP data being fed into KYB workflows through the portal, but this is downstream of partner sources, not an endpoint. Kausate's public API doesn't expose sanctions/PEP either — treat it as a separate vendor concern (ComplyAdvantage, OpenSanctions, Refinitiv) and bind it on your side. The registry calls move to Kausate; the screening calls stay where they are.

---

## 6. Status mapping table

Kyckr has multiple status enums depending on resource. They all collapse onto Kausate's six top-level `status` strings (see `../status-and-dedup.md`).

### Filing order — Kyckr V1 `statusField` (numeric) → Kausate `status`

| Kyckr | Kausate `status` | HTTP |
|---|---|---|
| `1` Pending | `running` | 200 (immediate) |
| `2` Ready | `completed` | 200 (delivered via webhook) |
| `3` Ready | `completed` | 200 |
| `21` Ready | `completed` | 200 |
| `5` Failed | `failed` | 200 (delivered with `error` populated) |
| `9` Cancelled | `canceled` | 200 |

### Filing order — Kyckr V2 (string) → Kausate `status`

| Kyckr V2 | Kausate `status` |
|---|---|
| `Pending` | `running` |
| `Complete` | `completed` |
| `Failed` | `failed` |

### UBO Verify tree — Kyckr V1 `statusField` → Kausate `status`

(See §5.5 — full table reproduced here for completeness.)

| Kyckr UBO `statusField` | Kausate `status` |
|---|---|
| `IN_PROGRESS` | `running` |
| `SUCCESS` | `completed` |
| `NO_SHAREHOLDERS` | `completed` (empty graph) |
| `INSUFFICIENT_MAX_LAYERS` | `completed` (truncated graph) |
| `COMPANY_NOT_FOUND` | `failed` |
| `REGISTRY_NOT_AVAILABLE` | `failed` / `timedOut` |
| `INSUFFICIENT_CREDITS` | `failed` (402-class) |
| `INSUFFICIENT_MAX_CREDIT_COST` | — (no analogue) |
| `ORDER_NOT_FOUND` | 404 |
| `SERVICE_UNAVAILABLE` | `failed` |

### `responseCodeField` (Kyckr V1 envelope) → Kausate

Kyckr V1 packs error semantics into a numeric `responseCodeField` inside the same envelope as the success payload. Kausate uses HTTP status + an error body. Mapping:

| Kyckr code | Name | Kausate HTTP / body |
|---|---|---|
| `100` | SUCCESS | 200 / `status: "completed"` |
| `102` | NO_RESULTS | 200 with empty `result.searchResults` |
| `104` | REGISTRY_NOT_AVAILABLE | 200 with `status: "failed"` (or `timedOut`); `error.message` carries the registry |
| `105` | COMPANY_NOT_FOUND | 404 or 200 with empty `result` (for search) |
| `106` | INVALID_REGISTRATION_AUTHORITY | (gone — no `RegAuth`) |
| `107` | PARAMETER_NOT_SUPPORTED | 422 `extra_forbidden` |
| `109` | TOO_MANY_MATCHES | 200 with truncated `searchResults` — refine the query |
| `110` | COMPANY_NUMBER_NOT_VALID_FORMAT | 422 |
| `111` | PRODUCT_NOT_AVAILABLE | 422 — check `/v2/platform/jurisdictions` for capability matrix |
| `112` | INVALID_ARGUMENT | 422 |
| `402` | INSUFFICIENT_FUNDS | 402 |
| `500` | SERVER_ERROR | 500 |

Kausate also adds statuses Kyckr doesn't model: `terminated` (forced termination, rare) and `timedOut` (workflow-level 7-day timeout, also registry-side timeouts). Treat `terminated` like `failed`; treat `timedOut` as retryable.

---

## 7. Errors

Kyckr V1 errors come back as the **same envelope as successes**, with `responseCodeField` carrying the failure code and the resource block (`companiesField`, `ownershipTreeField`, ...) either omitted, empty, or carrying its own `statusField`. There is no separate `error` object; you discriminate on `responseCodeField`. V2 returns a thinner envelope (`{ correlationId, customerReference, timeStamp, details, data }`) where `details` carries a human string like `"Success"` — the full V2 error enum isn't published.

Kausate uses HTTP status + a FastAPI-style error body:

```json
{ "detail": "Company not found" }
```

For async orders, the per-order `status` (`failed` / `terminated` / `timedOut`) carries the outcome, and `error: { message, code? }` carries the detail. **You discriminate on HTTP status + top-level `status`, not on a numeric code inside the envelope.**

Common Kausate errors and their Kyckr equivalents:

| Situation | Kyckr | Kausate |
|---|---|---|
| Bad parameter name | `responseCodeField: 107` | `422` with `{ "detail": [{ "loc": [...], "msg": "...", "type": "extra_forbidden" }] }` |
| Bad parameter value | `responseCodeField: 112` | `422` |
| Insufficient funds | `responseCodeField: 402`, HTTP 402 | HTTP 402 |
| Server error | `responseCodeField: 500`, HTTP 500 | HTTP 500 |
| Order not found | UBO `statusField: ORDER_NOT_FOUND`, HTTP 403 | HTTP 404 |
| Bad path | HTTP 404 | HTTP 404 |
| Sending `customerId` in body instead of header | (n/a) | HTTP 422 `extra_forbidden` |

Full error reference: `../errors-and-anti-patterns.md`.

---

## 8. Migration footguns

Things that bite when porting code mechanically:

- **Lowercase jurisdiction codes.** `GB` → `gb`, `DE` → `de`, `US-TX` → `us-tx`. A find-and-replace pass on the ISO codes is usually safe.
- **`RegAuth` dies.** Anywhere your code branches on DE / CA registration authority (passing `RegAuth=B`/`HRB`/`ON`, echoing `registrationAuthorityCodeField`), drop the branch. Kausate disambiguates at search time; `kausateId` already encodes the registry.
- **Italy native-id switch (Kyckr 2025-03-18).** If you updated your Kyckr IT integration to switch from Codice Fiscale to REA (`{province}{digits}`, e.g. `MI123456`), you already hold the correct native id for Kausate — REA is what Kausate's `companyNumber` expects for IT. If you didn't switch, do it now: Codice Fiscale is non-unique and profile/filing calls keyed on it can fail.
- **24h cache (Kyckr) vs same-day dedup (Kausate) — different semantics.** Kyckr's documented cache is *within UBO Verify retries*: enhanced profiles purchased in the first walk are reused for free in a continuation within 24h. No documented dedup on plain profile/filing endpoints. Kausate's **same-day dedup** is broader: any second `(kausateId, workflow, customerReference, customerId)` on the same UTC day returns the existing `orderId`. Pass `bypassCache: true` for a fresh call. **Don't port retry-with-jitter loops verbatim** — on Kausate the retry dedups unless you change a key or set `bypassCache`.
- **V2 `kyckrId` is volatile; `kausateId` is durable.** Kyckr V2 workflow guide: *"This format may change and should not be persisted."* If you persisted `kyckrId` anyway, wipe that column and re-derive via search. `kausateId` is the opposite — persist it, use it as your foreign key.
- **Polling cadence (1-min) → webhooks.** Don't port the "every minute, scan all in-flight orders" cron. Subscribe a webhook, treat the cron as reconciliation for stragglers — median completion-detection latency drops from ~30 s to ~0 s.
- **`productCodeField` literals differ per jurisdiction.** Kyckr's `productCodeField` values are registry-specific (`AnnualAccounts`, `Chronological`, `MM-1.1`, etc.) with no global catalogue. Don't literal-map them. Kausate's `documentType` is a small normalized enum — route on `documentType`; display `title` to the user.
- **Kyckr `OrderRef` ≠ Kausate `customerId`.** Kyckr's `OrderRef` → Kausate's `customerReference` (body field, ≤150 URL-safe chars). Kausate's **`customerId`** is different — an **end-customer / tenant identifier** sent via the **`X-Customer-Id` header**. Putting it in the body returns 422. See `../customer-correlation.md`.
- **No bulk endpoints on either side.** Client-side fan-out + reconciliation. There is no `/v2/companies/report/batch` on Kausate.
- **Refund policy.** Kyckr's docs are silent on whether `statusField=5` filings are credited back (the portal does it). Kausate refunds `AAKEVT` (monitor events) when every webhook retry fails; for data-order failures, check `/v2/analytics/breakdowns` rather than assuming a refund.

---

## 9. Before / after — two worked examples

### 9.1 Fetch a UK enhanced profile

**Before (Kyckr V1):**

```ts
// Step 1 — search to get codeField
const searchRes = await fetch(
  `https://rest.kyckr.com/core/company/search/GB/11655290`,
  { headers: { Authorization: process.env.KYCKR_KEY! } },
).then(r => r.json());

if (searchRes.responseCodeField !== "100") throw new Error("search failed");
const code = searchRes.companiesField[0].codeField;        // persist (code, "GB")

// Step 2 — enhanced profile (synchronous, slow — registry-bound)
const profileRes = await fetch(
  `https://rest.kyckr.com/core/company/profile/GB/${code}`,
  { headers: { Authorization: process.env.KYCKR_KEY! } },
).then(r => r.json());

const officials = profileRes.officialsField;
const shareholders = profileRes.shareholdersField;
```

**After (Kausate):**

```ts
// Step 1 — translate native id → kausateId (cache the mapping in your DB)
const search = await kausate.POST("/v2/companies/search/sync", {
  body: { companyNumber: "11655290", jurisdictionCode: "gb" },
});
const kausateId = search.data.result.searchResults[0].kausateId;
// Persist (companyNumber="11655290", jurisdictionCode="gb", kausateId) — durable.

// Step 2 — place an async report order; result arrives on the webhook
const order = await kausate.POST("/v2/companies/report", {
  body: { kausateId, customerReference: "kyc-case-12345" },
  headers: { "X-Customer-Id": "tenant-789" },
});
// order.data.orderId — store; you'll match the webhook delivery on this
```

```ts
// Step 3 — webhook receiver (Next.js — see ../async-webhooks.md for the full pattern)
export async function POST(req: Request) {
  const auth = req.headers.get("authorization");
  if (auth !== `Bearer ${process.env.KAUSATE_WEBHOOK_SECRET}`) {
    return new Response("forbidden", { status: 403 });
  }
  const payload = await req.json();

  // Ack within 5s, then queue the real work
  void queue.enqueue("kausate.completed", payload);
  return new Response(null, { status: 202 });
}

// Worker side
async function handle(payload: ExecutionResponse) {
  if (payload.status !== "completed") return surfaceError(payload);  // handle all six statuses
  if (payload.result?.type !== "companyReport") return;
  const { officials, shareholders } = payload.result.companyReport;
  // ...
}
```

### 9.2 Order a UBO report (and handle disambiguation)

**Before (Kyckr V1 UBO Verify):**

```ts
// Create
let create = await fetch(
  `https://rest.kyckr.com/core/uboverify/create/GB/09657876?uboThreshold=25&maxCreditCost=50&maxLayers=5`,
  { method: "POST", headers: { Authorization: process.env.KYCKR_KEY! } },
).then(r => r.json());
let orderId = create.ownershipTreeField.orderIdField;

// Poll list (once-a-minute)
let tree: any;
while (true) {
  await sleep(60_000);
  tree = await fetch(`https://rest.kyckr.com/core/uboverify/list/${orderId}`,
    { headers: { Authorization: process.env.KYCKR_KEY! } }).then(r => r.json());
  if (tree.statusField !== "IN_PROGRESS") break;
}

// Disambiguation: if COMPANY_NOT_FOUND with candidates, pick one and re-create
if (tree.statusField === "COMPANY_NOT_FOUND" && tree.reasonForNonContinuationField?.candidatesField) {
  const chosen = pickCandidate(tree.reasonForNonContinuationField.candidatesField);
  create = await fetch(
    `https://rest.kyckr.com/core/uboverify/create/GB/09657876?continuationKeys=${chosen.continuationKeyField}`,
    { method: "POST", headers: { Authorization: process.env.KYCKR_KEY! } },
  ).then(r => r.json());
  // ... back to poll loop
}

const ubos = tree.ultimateBeneficialOwnersField;
```

**After (Kausate):**

```ts
// Step 1 — disambiguate at search (NOT mid-walk)
const search = await kausate.POST("/v2/companies/search/sync", {
  body: { companyNumber: "09657876", jurisdictionCode: "gb" },
});
const kausateId = search.data.result.searchResults[0].kausateId;

// Step 2 — place the shareholder-graph order
const order = await kausate.POST("/v2/companies/shareholder-graph", {
  body: { kausateId, maxDepth: 5, enriched: true, customerReference: "ubo-case-1" },
  headers: { "X-Customer-Id": "tenant-789" },
});
// Webhook delivers result.type === "shareholderGraph" — no polling loop, no continuation dance
```

What you delete: polling loop, `continuationKeys` retry, `uboThreshold` param (filter on `result.ubos[].rolledUpPercentage` yourself), `maxCreditCost` cap (budget via `maxDepth` + analytics), RFNC routing table.

What you add: a webhook handler with a `shareholderGraph` branch.

---

## 10. Reference index

- **Auth, version pinning, generated client** — `../auth-versioning.md`
- **Endpoint inventory + native-id → `kausateId` translation** — `../endpoints.md`
- **Async lifecycle + webhook receiver pattern** — `../async-webhooks.md`
- **All six order statuses + same-day dedup + `bypassCache`** — `../status-and-dedup.md`
- **Document retrieval (two-step flow, pre-signed URLs)** — `../documents.md`
- **Change monitoring (sources, cron, event taxonomy)** — `../monitors.md`
- **Per-jurisdiction native ID formats** — `../identifiers.md`
- **`customerReference` vs `customerId` semantics** — `../customer-correlation.md`
- **Live OpenAPI spec** (auth required) — `https://api.kausate.com/openapi.json`

If anything here conflicts with `openapi.json`, the spec wins.
