# Migrating from Topograph to Kausate

Topograph is the closest architectural twin among Kausate's competitors — both REST, header-keyed, async with webhooks. Once you've renamed headers and translated identifiers, **the migration is mostly fanning one multi-`dataPoints` POST out into N per-product orders**, plus swapping a Svix HMAC verifier for a shared-secret header check.

If anything here disagrees with `https://api.kausate.com/openapi.json`, the spec wins.

---

## 1. Mental-model translation

Topograph models company data as **one multi-purpose POST**: `POST /v2/company { countryCode, id, dataPoints: [...] }` returns a single `requestId` whose `dataStatus` tracks per-dataPoint progress; you poll one URL until every dataPoint is terminal.

Kausate is **one endpoint per data product**: `report`, `finance`, `ubo`, `shareholder-graph`, `documents/list`, `documents`, `prefill`. Each is its own async order with its own `orderId`; webhooks are primary, polling is fallback. A Topograph call with three dataPoints becomes three parallel Kausate POSTs correlated by a shared `customerReference`. The credit cost is the same — only the wire shape and completion plumbing change.

---

## 2. Endpoint mapping

| Topograph | Kausate | Notes |
|---|---|---|
| `GET /v2/search?country=&query=` | `POST /v2/companies/search` (async) or `/search/sync` | Body-based; jurisdictionCode lowercase. |
| `GET /v2/search?stream=true` (SSE) | `GET /v2/companies/search/autocomplete` | Sync < 100 ms. No SSE in Kausate — see §7. |
| `POST /v2/company { dataPoints: [...] }` | Fan out — one POST per product. See §5. | |
| `POST /v2/company { mode: "onboarding" }` | `POST /v2/companies/prefill` | Always sync, < 200 ms. See §6. |
| `POST /v2/onboarding` (deprecated) | `POST /v2/companies/prefill` | |
| `POST /v2/company { requestId }` (cached re-read) | `GET /v2/companies/{family}/{orderId}` | Per-order polling. |
| `GET /v2/company/{requestId}` (poll) | Webhook on `POST /v2/webhooks` (primary) or `GET /v2/companies/{family}/{orderId}` (fallback) | See `../async-webhooks.md`. |
| `POST /v2/monitors`, `GET /v2/monitors/{id}/logs`, `DELETE /v2/monitors/{id}` | `POST /v2/monitors`, `GET /v2/monitors/{id}/events`, `DELETE /v2/monitors/{id}` | See `../monitors.md`. |
| `GET /v2/pricing?countryCode=` | (none) — post-hoc `GET /v2/analytics/summary` + dashboard | See §11. |
| `GET /v2/workspaces`, `/v2/workspaces/usage`, `/v2/billing/notifications/*` | (none — dashboard / API-key-per-tenant) | See §12. |
| Webhook `company.updated`, `monitor.notification` (Svix HMAC) | `ExecutionResponse` (order webhooks, shared-secret header) + `monitor.change_detected` (monitor webhooks, no HMAC) | See §10. |

---

## 3. Auth diff

```diff
- x-api-key: <key>                                 # Topograph
- x-topograph-workspace-id: <workspace-name>       # optional workspace attribution
+ X-API-Key: $KAUSATE_API_KEY
+ Kausate-Version: 2026-05-01                      # mandatory — pin every call
+ X-Customer-Id: <end-customer-id>                 # optional end-customer attribution
```

1. **`x-api-key` → `X-API-Key`.** Same model (org-scoped, header-only). Rotate via the dashboard.
2. **`x-topograph-workspace-id` has no Kausate equivalent.** Kausate API keys are org-scoped; there's no workspace tier between org and key. For per-tenant attribution use `X-Customer-Id` (see §12, `../customer-correlation.md`).
3. **`Kausate-Version: 2026-05-01` is mandatory.** Topograph has no versioning header — its OpenAPI evolves at-will. Kausate uses date-based versioning; skip the header and you get the org's default, which can shift. Pin per request and in your generated-client config. See `../auth-versioning.md`.

---

## 4. Identifier translation

```text
Topograph:  per-call  { countryCode: "FR", id: "552032534" }      # uppercase
            per-call  requestId                                    # server-issued, aggregates all dataPoints
Kausate:    once      kausateId  = "co_fr_..."                    # lowercase jurisdiction prefix
            per-call  orderId                                      # server-issued, per product
            per-call  customerReference                            # your value, used for cross-order correlation
```

Translate native ID → `kausateId` **once** via search, then cache the mapping:

```bash
# Topograph: (countryCode: "FR", id: "552032534")
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  -d '{"companyNumber":"552032534","jurisdictionCode":"fr"}'
# → result.searchResults[0].kausateId
```

```ts
async function resolveKausateId(country: string, nativeId: string) {
  const hit = mapping.get(`${country}:${nativeId}`);
  if (hit) return hit;
  const r = await kausate.POST("/v2/companies/search/sync", {
    body: { companyNumber: nativeId, jurisdictionCode: country.toLowerCase() },
  });
  const id = r.data.result.searchResults[0]?.kausateId;
  if (id) mapping.set(`${country}:${nativeId}`, id);
  return id;
}
```

**Lowercase quirk.** Topograph uses `"FR"`, `"DE"`. Kausate uses `"fr"`, `"de"`. Same ISO 3166-1 alpha-2; just `.toLowerCase()`.

Topograph's `SearchResult.country` can return any of 261 ISO codes plus US state codes (`US-FL US-IL US-MA US-NJ US-NY US-TX US-DE US-WA US-GA US-NV US-PA`) and `TEST` because graph traversal crosses jurisdictions. Kausate jurisdictions are restricted to supported ones (50+); see `GET /v2/platform/jurisdictions` and `../identifiers.md`.

---

## 5. Per-dataPoint migration

Each Topograph dataPoint maps to a different Kausate endpoint (or a slice of one).

### 5.1 `company` → `POST /v2/companies/report`

```bash
# Topograph
POST /v2/company { countryCode:"FR", id:"552032534", dataPoints:["company"] }
# poll until dataStatus.dataPoints.company.status === "succeeded"
# → response.company (CompanyDTO)

# Kausate
POST /v2/companies/report { kausateId, customerReference }
# → { orderId, status:"running" }   webhook fires with result.type == "companyReport"
```

| Topograph `CompanyDTO` | Kausate `companyReport.result` |
|---|---|
| `legalName`, `legalNameInEnglish` | `basicInformation.legalName`, `legalNameInEnglish` |
| `commercialNames`, `legacyLegalNames` | `basicInformation.tradingNames`, `historicalNames` |
| `status.statusDetails.status` (`ACTIVE / UNDER_INSOLVENCY_PROCEEDING / CLOSED / UNKNOWN`) | `basicInformation.companyStatus` (per-jurisdiction enum) |
| `legalForm.standardized`, `legalForm.iso20275Code` | `basicInformation.legalForm` |
| `taxId.value` (VAT, **no country prefix** — `27443061841`) | `identifiers.vat` (per-jurisdiction format) |
| `taxId.verification.consultationNumber` (VIES legal proof) | (no equivalent) |
| `capital`, `activities`, `activityDescription`, `legalAddress` | `basicInformation.capital`, `industryClassifications`, `activityDescription`, `addresses.registered` |
| `legalRepresentatives` (separate Topograph dataPoint) | Same `companyReport` payload — see §5.2 |

### 5.2 `legalRepresentatives` → folded into `companyReport.result`

Topograph treats legal reps as a separate dataPoint. In Kausate they're already part of `companyReport.result.legalRepresentatives[]` — **there is no separate endpoint to order them**.

| Topograph `LegalRepresentativeDTO` | Kausate `legalRepresentatives[]` |
|---|---|
| `entityId` (cross-list dedupe key) | (not guaranteed cross-list — name-match if you need it) |
| `role.iso5009Code`, `role.standardized` | `role` |
| `type` (`individual` / `company`) | `subjectType` |
| `representationMode` (`sole` / `joint`, `minimumSignatories`, `namedCoSigners[]`) | `representationMode` (where the registry exposes it) |
| `representedBy` (human acting for corporate director) | `representedBy` |
| `startDate`, `endDate` | `startDate`, `endDate` |

`otherKeyPersons` (board members, secretaries, auditors, compliance officers, …) is similarly folded into `report` — no separate endpoint.

### 5.3 `shareholders` → `POST /v2/companies/shareholder-graph` with `maxDepth: 1`

Topograph's `shareholders` returns one layer of direct holders. Kausate has no single-layer shareholder endpoint; the equivalent is `shareholder-graph` with `maxDepth: 1`.

```bash
POST /v2/companies/shareholder-graph { kausateId, maxDepth:1, customerReference }
```

| Topograph `ShareholderDTO` | Kausate node in `shareholderGraph.result.nodes[]` |
|---|---|
| `entityId`, `type` | `nodeId`, `subjectType` |
| `sharePercentage`, `numberOfShares` | `sharePercentage`, `numberOfShares` |
| `nominalCapitalHeld` (*valore nominale* / *Stammkapital*) | `nominalCapitalHeld` |
| `paidInAmount` (*valore versato* / *eingezahltes Kapital*) | `paidInAmount` |

### 5.4 `ubo` (`ultimateBeneficialOwners`) → `POST /v2/companies/ubo`

```bash
POST /v2/companies/ubo { kausateId, customerReference }
# webhook → result.type == "uboReport"
```

| Topograph `UltimateBeneficialOwnerDTO` / `ControlDTO` | Kausate `uboReport.result.beneficialOwners[]` |
|---|---|
| `control.types` (`ownership-of-shares` / `voting-rights` / `appoint-and-remove-directors` / `significant-influence-or-control`) | `controlTypes` |
| `control.description` (human-readable) | `controlDescription` |
| `control.details[].type` (`shares` / `voting-rights`) | `details[].controlType` |
| `control.details[].percentageRange.{lower,upper}`, `percentageValue` | `percentage` (exact) or `percentageRange` (banded) |
| `control.details[].nature` (`direct` / `indirect`) | `nature` |

For UBO chains crossing borders, use `shareholder-graph` with deeper `maxDepth` and read UBO flags off the nodes.

### 5.5 `subsidiaries` → `POST /v2/companies/shareholder-graph` (downstream)

Both APIs use graph traversal here. Topograph's `subsidiaries` is **strictly downstream** (one layer). Kausate's `shareholder-graph` is **bidirectional in one call**.

```bash
POST /v2/companies/shareholder-graph { kausateId, maxDepth:1, customerReference }
```

If you don't want upstream nodes, filter edges by direction client-side.

| Topograph `SubsidiaryDTO` | Kausate downstream edge / node |
|---|---|
| `sharePercentage` (direct) | edge `percentage` |
| `totalSharePercentage` (consolidated, when computable) | node `totalOwnershipPercentage` |
| `votingRightsPercentage` | `votingRightsPercentage` |
| `acquisitionDate`, `endDate` | `acquisitionDate`, `endDate` |

### 5.6 `financials` → `POST /v2/companies/finance`

Topograph doesn't have a `financials` dataPoint per se — financial data arrives via the `documents` dataPoint (`AvailableDocumentDTO.type: "financial_statement"`), then a beta extraction pass produces `ExtractedFinancialDataDTO` + `FinancialAnalysisDTO`. Kausate has a dedicated endpoint:

```bash
POST /v2/companies/finance { kausateId, customerReference }
# webhook → result.type == "financials"
```

| Topograph `ExtractedFinancialDataDTO` / `FinancialAnalysisDTO` | Kausate `financials.result` |
|---|---|
| `fiscalYear`, `approvalDate`, `currency` | `fiscalYear`, `approvalDate`, `currency` |
| `accountingStandard` (`IFRS / French GAAP / US GAAP / Swiss GAAP / Lux GAAP / Other`) | `accountingStandard` |
| `statementType` (`consolidated / simplified`) | `statementType` |
| `incomeStatement`, `balanceSheet` | `incomeStatement`, `balanceSheet` (per-jurisdiction shape) |
| `analysis.healthScore`, `overallRisk`, `ratios.*`, `insights`, `strengths`, `concerns`, `trends` | (no equivalent — Kausate exposes raw data, not AI risk analysis) |

If you depend on Topograph's `healthScore` / risk analysis, keep that layer on your side and feed Kausate's raw financials into it.

### 5.7 `documents` → `POST /v2/companies/documents/list` → `POST /v2/companies/documents`

Both APIs are two-phase. Topograph's phase 1 (`availableDocuments`) is free; Kausate's `documents/list` is a billed order. See `../documents.md`.

```bash
# Topograph
POST /v2/company { dataPoints:["availableDocuments"] }            # free
POST /v2/company { documents:["1c932de4-..."] }                   # billed; → DocumentsDTO buckets

# Kausate
POST /v2/companies/documents/list { kausateId, customerReference } # billed; → documents[] flat
POST /v2/companies/documents { kausateId, kausateDocumentId, customerReference } # → downloadLink + expiresAt
```

| Topograph | Kausate |
|---|---|
| `AvailableDocumentDTO.id` | `kausateDocumentId` |
| `DocumentWithUrlDto.url` (signed, **15-min lifetime**); re-issue **free within 24 h** via `GET /v2/company/{requestId}` | `result.downloadLink` + `result.expiresAt` — honor the expiry; re-fetch is a billed order |
| `pdfUrl` (auto-converted PDF when original is XML) | (no auto-conversion — convert client-side if needed) |
| Response **buckets** (`tradeRegisterExtract`, `financialStatements[]`, `articlesOfAssociation[]`, `ultimateBeneficialOwnersCertificate`, `officialPublications[]`, `annualReturns[]`, `certificateOfGoodStanding`, `otherDocuments[]`, `lastFiscalYearFinancialStatement`) | Flat `documents[]`, discriminated by `documentType` — bucket client-side if your UI needs it |

**Document type vocabulary** — Topograph (11 values) vs Kausate (closest):

| Topograph `type` | Kausate `documentType` |
|---|---|
| `uncertified_trade_register_extract`, `certified_trade_register_extract` | `currentExtract` |
| `trade_register_history` | `chronologicalExtract` |
| `financial_statement` | `annualAccounts` |
| `ubo_extract` | `beneficialOwnersDetails` |
| `annual_return` | `annualReturn` |
| `official_publication` | `officialFilings` |
| `status` (articles of incorporation) | `articlesOfAssociation` |
| `certificate_of_good_standing` | `registrationDetails` (varies) |
| `other`, `unknown` | (filter — `documentType` can be `null`) |

Verify per-jurisdiction; precise mapping depends on what each registry publishes.

### 5.8 `companyProfile` → fan out

`companyProfile` is a Topograph legacy alias that expands to `company + legalRepresentatives + otherKeyPersons + establishments` (best-effort per country). Kausate's `companyReport` already includes the latter three where the registry exposes them, so:

```ts
const ref = `kyc-case-${caseId}`;
await Promise.all([
  kausate.POST("/v2/companies/report",  { body: { kausateId, customerReference: ref } }),
  kausate.POST("/v2/companies/ubo",     { body: { kausateId, customerReference: ref } }), // if needed
  kausate.POST("/v2/companies/finance", { body: { kausateId, customerReference: ref } }), // if needed
]);
```

### 5.9 `graphInteractive` + `graphContinueFromNodeIds` → full graph in one call

**Behavioral diff.** Topograph's `graphInteractive: true` fetches only the entry node and returns direct shareholders as `budget_truncated` placeholder chips; each click feeds back a `nodeId` via `graphContinueFromNodeIds`, fanning out the graph one step at a time with per-step billing. Once interactive, every continuation is auto-flagged interactive — the client-supplied value is ignored.

**Kausate has no click-by-click expansion.** `shareholder-graph` is "one full graph in one call." Intended pattern:

1. One `shareholder-graph` order at the `maxDepth` you actually need (1–7).
2. Cache the full result client-side.
3. Expand/collapse the cached graph in your UI; no server round-trips per click.

```bash
POST /v2/companies/shareholder-graph { kausateId, maxDepth:3, enriched:true, customerReference }
```

If you need to go deeper than your initial fetch, place another order at a larger `maxDepth`. Kausate's same-day dedup will charge correctly (same `(kausateId, workflow, customerReference, customerId)` returns cached).

### 5.10 `agenticEnrichment` (Belgium €0.50) → `enriched: true` on shareholder-graph

Topograph's `agenticEnrichment: true` is **Belgium-only**, **fixed €0.50 per request**, AI-enriches legal representatives' birthDate / nationality / residence by reading publications. It triggers `dataStatus.<dataPoint>.status: "enriching"`.

Kausate's `enriched: true` on `shareholder-graph` is **governance enrichment of the graph itself** — different scope. There's no direct Kausate equivalent for BE legal-rep PII enrichment today. If you depend on it for Belgian KYB, plan to keep that source side-by-side or fall back to Kausate's documents flow (BE UBO extract).

---

## 6. `mode: verification | onboarding` translation

| Topograph | Kausate |
|---|---|
| `mode: "verification"` (default) — authoritative registers, retries up to 1 h on transient errors | Standard async endpoints (`report`, `ubo`, `finance`, …) |
| `mode: "onboarding"` — fast/cheap tier, **hard 10 s per-dataPoint deadline**, errors with `onboarding_timeout` / `fast_source_unavailable` | `POST /v2/companies/prefill` — always sync, < 200 ms typical; index hit with live fallback |
| `authoritative: true` flag in response (`verification`) | (no equivalent flag) — `companyReport` is always live |
| `authoritative` **silently flipped to `false`** in `onboarding` if fast source isn't authoritative | (no equivalent) — endpoint distinguishes, not a flag |

**Footgun.** Topograph's `mode: "onboarding"` silently degrades `authoritative: false`. If your existing code gates KYB decisions on `response.authoritative`, **that signal goes away** — Kausate distinguishes at the endpoint level (`prefill` = fast tier, `report` = registry-live), not via a per-call flag.

Staged-verification pattern translates cleanly: `prefill` for the instant render, `report` for the KYB gate.

```ts
// Topograph: check the flag, fire a second call if degraded
// Kausate: pick the endpoint based on what you need
const fast = await kausate.POST("/v2/companies/prefill", { body: { kausateId } });
showInstantly(fast.data.result);
await kausate.POST("/v2/companies/report", {
  body: { kausateId, customerReference: ref },
}); // result lands on webhook — gate KYB on that
```

---

## 7. Search

```text
Topograph: GET /v2/search?country=FR&query=...              (sync, ranked list)
           GET /v2/search?stream=true                        (SSE: progress / complete / error events)
Kausate:   POST /v2/companies/search                         (async with webhook)
           POST /v2/companies/search/sync                    (sync, 5 s typical, 300 s cap)
           GET /v2/companies/search/autocomplete             (type-ahead, < 100 ms)
           GET /v2/companies/search/person                   (by legal rep)
```

Body fields on the POST: `companyName`, `companyNumber`, `advancedQuery`, `jurisdictionCode`, `customerReference`. Use `companyNumber` when you have a native registry number to avoid fuzzy matching.

**SSE streaming.** Topograph emits a `progress` event once per upstream source (e.g. DE fires twice: Unternehmensregister then Handelsregister) so type-ahead UIs can populate inside the first second. Kausate doesn't stream — use `GET /v2/companies/search/autocomplete` (sync sub-100 ms) for type-ahead. The receiver complexity drops considerably.

**MatchType ordering.** Topograph response items carry `matchReason.matchType` sorted `id > exactLegalName > partialId > default`. Kausate's results are pre-ranked by relevance (registry-side + Kausate-side scoring); no separate `matchType` enum is exposed.

**Country params.** Topograph's request `country` parameter is restricted to the 40 supported jurisdictions, but `SearchResult.country` returns the broad enum (261 ISO codes + US state codes + `TEST`) because graph traversal crosses borders. Kausate's `jurisdictionCode` is similarly restricted to supported jurisdictions; cross-border response data inherits its own jurisdiction code.

---

## 8. Polling vs webhooks (structural diff)

| Topograph | Kausate |
|---|---|
| **One `requestId`** aggregates all dataPoints | **N `orderId`s**, one per product |
| Poll `GET /v2/company/{requestId}` until every `dataStatus.dataPoints[*]` is terminal (recommended 3 s interval, max 10 req/s) | Webhook delivery per order (one per `orderId`, terminal only). Fallback poll `GET /v2/companies/{family}/{orderId}` |
| `company.updated` fires **progressively** as each dataPoint completes — same `requestId`, multiple deliveries | One delivery per order at terminal completion. Multiple orders share `customerReference` for correlation |

**Aggregation footgun.** If your Topograph receiver aggregated state per `requestId`, switch to aggregating per `customerReference` (your value) in Kausate. Each delivery is terminal for one product; you decide when "the case is ready" by counting expected orders against landed webhooks.

See `../async-webhooks.md` for the receiver checklist (idempotent on `orderId`, 30 s ack timeout, 50 retries with exponential backoff to 4 h max interval).

---

## 9. Monitors

### Categories — verbatim identical

Topograph: `status, address, ownership, financial, legalRepresentatives, other, disappeared`.
Kausate:   `status, address, ownership, financial, legalRepresentatives, other, disappeared`.

Routing logic that switches on Topograph's category ports 1:1.

### Endpoint mapping

| Topograph | Kausate |
|---|---|
| `POST /v2/monitors { companyId, countryCode }` | `POST /v2/monitors { kausateId, sources, scheduleCron, webhookUrl, categoriesFilter, eventCodesFilter, autoDeactivateCategories }` |
| `GET /v2/monitors` (cursor `after`, `first`) | `GET /v2/monitors` (cursor pagination) |
| `GET /v2/monitors/{id}` | `GET /v2/monitors/{id}` |
| `DELETE /v2/monitors/{id}` | `DELETE /v2/monitors/{id}` |
| `GET /v2/monitors/{id}/logs` (`changeDetected`, `changeCategories`, `fetchStatus`: `success / not_found / error / error_final`) | `GET /v2/monitors/{id}/events` (persisted including filtered-out; supports `replay`) |

### Cron model

Topograph fetches **daily** for every monitor — no schedule control. Kausate uses an explicit `scheduleCron` (5-field, UTC) that drives `per_company_pull` sources; `global_feed` sources run centrally on upstream publication.

### Signing — biggest receiver-side change

| Topograph | Kausate |
|---|---|
| **Svix HMAC** — `svix-id`, `svix-timestamp`, `svix-signature` headers; secret `whsec_…`; verify with Svix SDK | Order webhooks: **shared-secret custom header** (you choose). Monitor webhooks: **no HMAC, no custom headers today** — use an unguessable HTTPS path or front with a reverse-proxy |

### Webhook payload — different cardinality

Topograph (`monitor.notification`) fires **once with an array** of changed categories:

```json
{
  "type": "monitor.notification",
  "monitorId": "cmfzlu1dt000j12ldeqrfszq0",
  "companyId": "123456789",
  "countryCode": "DE",
  "timestamp": "2026-03-20T02:00:00Z",
  "changeCategories": ["status", "address"],
  "monitorHasBeenDeactivated": false
}
```

Kausate (`monitor.change_detected`) fires **once per event** — N changes → N deliveries:

```json
{
  "event": "monitor.change_detected",
  "monitor_id": "5d4241a1-...",
  "kausate_id": "co_de_...",
  "event_id": "b2fa69b8-...",
  "event_code": "INSOLVENCY_DECISION",
  "category": "status",
  "severity": "info",
  "detected_at": "2026-05-04T22:00:00Z",
  "before": null,
  "after": { /* publication payload or new value */ },
  "metadata": { "source": "...", "jurisdiction": "de", "api_url": "..." }
}
```

Key differences:
- Kausate uses **snake_case** on monitor webhooks; order webhooks use camelCase. **Don't share a single deserialization model across both channels.**
- Idempotency key is `event_id` (not `monitor_id`).
- Kausate adds `before` / `after` / `diff_path` so you can render specific value transitions.
- Kausate's `event_code` is more granular than `category` (`INSOLVENCY_OPENED` vs `INSOLVENCY_DECISION`). Route on `category` for fan-out, `event_code` for specific transitions.

See `../monitors.md` for sources discovery, `event_code` taxonomy, filtering, and auto-deactivation.

---

## 10. Webhooks — Svix HMAC → shared-secret header

The biggest receiver-side change. Topograph signs every webhook with Svix HMAC; Kausate order webhooks authenticate via a header you set at subscription.

**Before — Topograph receiver:**

```ts
import { Webhook } from "svix";
import express from "express";

const app = express();
const wh = new Webhook(process.env.TOPOGRAPH_WEBHOOK_SECRET!); // whsec_...

app.post("/webhooks/topograph", express.raw({ type: "*/*" }), (req, res) => {
  let payload;
  try {
    payload = wh.verify(req.body, {
      "svix-id":        req.header("svix-id")!,
      "svix-timestamp": req.header("svix-timestamp")!,
      "svix-signature": req.header("svix-signature")!,
    });
  } catch { return res.status(401).end(); }
  res.status(200).end();
  void enqueue(payload);
});
```

**After — Kausate receiver:**

Subscribe with a custom header:

```bash
curl -X POST https://api.kausate.com/v2/webhooks \
  -H "X-API-Key: $KAUSATE_API_KEY" -H "Kausate-Version: 2026-05-01" \
  -d '{
    "name": "Order completions",
    "url": "https://your-app.example.com/webhooks/kausate",
    "customHeaders": { "Authorization": "Bearer '"$KAUSATE_RECEIVER_SECRET"'" }
  }'
```

Verify the header in your handler:

```ts
import express from "express";

const app = express();
const SECRET = process.env.KAUSATE_RECEIVER_SECRET!;

app.post("/webhooks/kausate", express.json(), (req, res) => {
  if (req.header("Authorization") !== `Bearer ${SECRET}`) return res.status(401).end();
  const payload = req.body; // ExecutionResponse — { orderId, status, result, ... }
  res.status(200).end();
  void enqueueIdempotent(payload.orderId, payload); // dedup on orderId
});
```

Differences worth flagging:
- **No raw-body handling.** No HMAC → plain `express.json()` is fine.
- **Idempotency key is `orderId`** (or `event_id` for monitors). Kausate retries up to 50× (exp backoff, 1 s → 4 h max); same-day dedup can also produce duplicates.
- **30 s ack timeout** (Topograph: retries on non-2xx for up to 3 days).
- Strip the Svix SDK entirely. Replay protection is via `orderId` / `event_id` idempotency, not signature.

---

## 11. Pricing surface diff

| Topograph | Kausate |
|---|---|
| `GET /v2/pricing?countryCode=FR` returns `blocks[].modes.{verification, onboarding}`, `pricingMode: fixed \| variable`, `includedDocuments[]`, `monitoring` (zero-cost auto-monitored) | **No pre-query pricing endpoint.** Post-hoc only. |
| Prices in **credits (cents)**; "may vary by subscription plan" | Per-SKU in dashboard; per-call cost via `GET /v2/analytics/summary` |
| `maxBudget`, `profileMaxBudget`, `graphMaxBudget` — hard caps; fail with `budget_exceeded` | (no per-call budget knobs — org-level alerts in dashboard) |
| `pricingMode: variable` (Hungary etc.) — resolved at request time | All workflows priced per-SKU at completion; no pre-quote |
| Monitoring auto-billed at zero cost on zero-cost blocks | `AAKMON` (1 credit per scheduled tick) + `AAKEVT` (1 credit per delivered event, refunded if all retries fail) — see `../monitors.md` |

**Strategy advice:**
- Drop any code branching on `GET /v2/pricing`. The endpoint split (`prefill` cheap, async endpoints registry-live) replaces the per-call decision.
- If you used `maxBudget` to cap spend on variable-priced countries, enforce a tighter org-level cap in the dashboard or pre-screen which `kausateId`s your end-customers can query.
- Treat every call as billable; monitor spend via `GET /v2/analytics/summary` and `GET /v2/analytics/breakdowns?groupBy=sku`.
- 24 h dedup exists in Kausate too, but the key is `(kausateId, workflow, customerReference, customerId, UTC-day)` — see §16 and `../status-and-dedup.md`.

---

## 12. Workspaces

Topograph supports multiple workspaces per account:

- `GET / POST / PATCH / DELETE /v2/workspaces`, `GET /v2/workspaces/usage`
- Per-workspace attribution via `x-topograph-workspace-id` header
- Per-workspace billing notifications: `low_balance` (`warning / critical / depleted`), `high_usage` (`warning / critical`, global vs per-workspace), `auto_topup` (`succeeded / failed`); `GET / PATCH /v2/billing/notifications/config`, `GET /v2/billing/notifications/recent`, per-workspace overrides
- Each event carries a `dedupKey` over (kind, identifier, account, period)

**Kausate is org-scoped via API key.** No workspace tier, no header, no per-workspace notification endpoints.

Migration patterns:

1. **One Kausate API key per Topograph workspace** — each behaves as its own "tenant."
2. **`X-Customer-Id` header for end-customer attribution within a key** — see `../customer-correlation.md`.
3. **`GET /v2/analytics/breakdowns?groupBy=customerId`** replaces `GET /v2/workspaces/usage`.
4. **Org-level alerts in the dashboard** replace per-workspace billing notifications (low-balance, high-usage, auto-topup).

Topograph's billing-event `dedupKey` has no equivalent — Kausate doesn't emit billing webhooks today.

---

## 13. Documents — client-side bucketing helper

Mapping covered in §5.7. If your UI depended on Topograph's `DocumentsDTO` buckets, rebuild them from the flat list:

```ts
const buckets = {
  tradeRegisterExtract:  docs.find(d => d.documentType === "currentExtract"),
  financialStatements:   docs.filter(d => d.documentType === "annualAccounts"),
  articlesOfAssociation: docs.filter(d => d.documentType === "articlesOfAssociation"),
  ultimateBeneficialOwners: docs.filter(d => d.documentType === "beneficialOwnersDetails"),
  officialPublications:  docs.filter(d => d.documentType === "officialFilings"),
  annualReturns:         docs.filter(d => d.documentType === "annualReturn"),
};
```

Reminder: Topograph re-issues a fresh signed URL **free within 24 h** via `GET /v2/company/{requestId}`. Kausate's phase-2 re-fetch is a **billed order** — persist the bytes after first download, not the link.

---

## 14. TEST country sandbox

**Topograph** ships a `countryCode: "TEST"` sandbox with special IDs: `RESOURCE_NOT_FOUND`, `INVALID_REQUEST`, `PROCESSING_FAILED`, `SERVICE_UNAVAILABLE` trigger the matching error responses; `GRAPH_*` prefixes return canned shareholder structures (simple, chained, circular, wide, mixed, deep, empty); all bundle a free Trade Register Extract.

**Kausate has no sandbox.** Every call hits production registries and consumes credits. Practical guidance:

- Use **small test budgets** during development; monitor `GET /v2/analytics/summary`.
- Pick representative real `kausateId`s as your test fixtures; persist the search → `kausateId` translation once.
- **Don't hit the live API in receiver tests.** Record one real `ExecutionResponse` per `result.type`; replay against your handler offline.
- For canned graph topologies that exercised your code paths, hand-curate a few real companies with the patterns you need (circular structures, deep chains, wide spreads).

---

## 15. Status / error mapping

```text
Topograph (per-dataPoint, keyed by name AND by document UUID):
  dataStatus.dataPoints.<name>.status:        "succeeded" | "in_progress" | "enriching" | "failed"
  dataStatus.documents.<doc-uuid>.status:     same enum
Kausate (per-order, six values — see ../status-and-dedup.md):
  order.status: "running" | "completed" | "failed" | "canceled" | "terminated" | "timedOut"
```

| Topograph `DataPointStatus` | Kausate `order.status` |
|---|---|
| `succeeded` | `completed` |
| `in_progress` | `running` |
| `enriching` (BE agentic enrichment) | `running` (no direct signal — see §5.10) |
| `failed` | `failed` / `terminated` / `timedOut` (three failure modes; check `error.code`) |

**Granularity diff.** Topograph reports per-dataPoint AND per-document status under one `requestId`. Kausate splits each into its own order, so resolution happens per-order. Walk each `customerReference`-correlated order independently.

| Topograph `DataPointError.code` | Kausate equivalent |
|---|---|
| `resource_not_found` | HTTP 404 at placement, or `failed` with `error.code` for unknown `kausateId` |
| `service_unavailable` | `failed` / `timedOut` |
| `processing_failed` | `failed` |
| `invalid_request` | HTTP 422 (Pydantic body validation) |
| `datapoint_not_supported` | HTTP 422 (jurisdiction doesn't support — check `GET /v2/platform/jurisdictions`) |
| `fast_source_unavailable` | (no equivalent — `prefill` has its own fallback chain) |
| `onboarding_timeout` | `timedOut` (rare on `prefill`; common on slow registries) |
| `insufficient_funds` | HTTP 402 |
| `budget_exceeded` | (no equivalent — no per-call budget caps) |

See `../errors-and-anti-patterns.md`.

---

## 16. Migration footguns

- **`companyProfile` can't mix with granular dataPoints in one Topograph request.** Kausate has no such constraint — each order is independent. Delete the workaround.
- **VAT in `taxId.value` excludes country prefix in Topograph** (`27443061841`, not `FR27443061841`). Kausate handles VAT per-jurisdiction via the relevant `identifiers.vat` field — verify the format per country (some prefixed, some not). See `../identifiers.md`.
- **`graphInteractive` is inherited and non-overridable in continuations.** Kausate has no continuation API — drop `graphInteractive` everywhere.
- **Dedup key differs.** Topograph: per-account, per-company, per-block, 24 h, **workspace-independent**. Kausate: same UTC day, same `(kausateId, workflow, customerReference, customerId)`. **`customerReference` is part of the key** — reuse it to deduplicate, vary it (or set `bypassCache: true`) to force a fresh call. See `../status-and-dedup.md`.
- **`mode: "onboarding"` silently degrades `authoritative: false`.** Kausate's `prefill` is a separate endpoint with no flag — gate at the endpoint, not the field.
- **`dataStatus` keyed by both dataPoint name AND document UUID** in Topograph. Discriminate on the wrapping `status`. Kausate has one `result.type` discriminator and one `status` field per order — much simpler.
- **`requestId` lifetime: cached "permanently" in Topograph** (no documented TTL). Kausate `orderId` is stable; the `result` payload has no separate TTL. Document `downloadLink`s are time-limited per `expiresAt`.
- **Switching workspaces in Topograph does NOT re-trigger billing.** In Kausate, varying `customerId` **DOES** count as a different dedup key — same call from a different `customerId` will re-bill. Use `customerReference` for explicit deduplication within a customer.
- **Lowercase jurisdictions everywhere.**

---

## 17. Before / after — three worked examples

### A. Full company onboarding (search → company + UBO → use)

**Topograph:**

```ts
const [{ country, id }] = await fetch(
  `https://api.topograph.co/v2/search?country=FR&query=${encodeURIComponent("BNP Paribas")}`,
  { headers: { "x-api-key": KEY } }).then(r => r.json());

const { requestId } = await fetch("https://api.topograph.co/v2/company", {
  method: "POST", headers: { "x-api-key": KEY, "Content-Type": "application/json" },
  body: JSON.stringify({ countryCode: country, id,
    dataPoints: ["company", "legalRepresentatives", "ultimateBeneficialOwners"] }),
}).then(r => r.json());

let result;
do {
  await new Promise(r => setTimeout(r, 3000));
  result = await fetch(`https://api.topograph.co/v2/company/${requestId}`,
    { headers: { "x-api-key": KEY } }).then(r => r.json());
} while (!Object.values(result.request.dataStatus.dataPoints)
  .every(s => s.status === "succeeded" || s.status === "failed"));
// use result.company, result.legalRepresentatives, result.ultimateBeneficialOwners
```

**Kausate:**

```ts
const search = await kausate.POST("/v2/companies/search/sync", {
  body: { companyName: "BNP Paribas", jurisdictionCode: "fr" },
});
const kausateId = search.data.result.searchResults[0].kausateId;

const ref = `kyc-case-${caseId}`;
await Promise.all([
  kausate.POST("/v2/companies/report", { body: { kausateId, customerReference: ref } }),
  kausate.POST("/v2/companies/ubo",    { body: { kausateId, customerReference: ref } }),
]);
// Both webhooks land under customerReference=ref. Aggregate in handler.
// (legalRepresentatives is inside companyReport.result — no separate order.)
```

### B. Subsidiary exploration

**Topograph** (interactive, pay-per-click):

```ts
const root = await fetch("https://api.topograph.co/v2/company", {
  method: "POST", headers: { "x-api-key": KEY, "Content-Type": "application/json" },
  body: JSON.stringify({ countryCode: "FR", id: "552032534", dataPoints: ["graph"], graphInteractive: true }),
}).then(r => r.json());

// On click — continue from a budget_truncated nodeId. graphInteractive is auto-inherited.
const next = await fetch("https://api.topograph.co/v2/company", {
  method: "POST", headers: { "x-api-key": KEY, "Content-Type": "application/json" },
  body: JSON.stringify({ graphContinueFromNodeIds: [clickedNodeId] }),
}).then(r => r.json());
```

**Kausate** (full graph, explore client-side):

```ts
await kausate.POST("/v2/companies/shareholder-graph", {
  body: { kausateId, maxDepth: 3, enriched: true, customerReference: ref },
});
// Webhook → result.type == "shareholderGraph"
// Cache result.nodes + result.edges. Implement expand/collapse over the cached graph.
// To go deeper later: another order at a larger maxDepth.
```

### C. Monitor migration (Svix HMAC → Kausate)

**Topograph receiver + create:**

```ts
import { Webhook } from "svix";
const wh = new Webhook(process.env.TOPOGRAPH_WEBHOOK_SECRET!);

app.post("/webhooks/topograph", express.raw({ type: "*/*" }), (req, res) => {
  const payload = wh.verify(req.body, {
    "svix-id": req.header("svix-id")!,
    "svix-timestamp": req.header("svix-timestamp")!,
    "svix-signature": req.header("svix-signature")!,
  });
  if (payload.type === "monitor.notification") {
    for (const cat of payload.changeCategories) route(cat, payload);
  }
  res.status(200).end();
});

await fetch("https://api.topograph.co/v2/monitors", {
  method: "POST", headers: { "x-api-key": KEY, "Content-Type": "application/json" },
  body: JSON.stringify({ companyId: "123456789", countryCode: "DE" }),
});
```

**Kausate receiver + create:**

```ts
// Receiver — no HMAC; monitor webhooks have no custom-header support today
app.post("/webhooks/kausate-monitors/<unguessable-segment>", express.json(), (req, res) => {
  const p = req.body;
  if (p.event !== "monitor.change_detected") return res.status(200).end();
  if (alreadyProcessed(p.event_id)) return res.status(200).end(); // idempotent on event_id
  markProcessed(p.event_id);

  switch (p.category) {
    case "disappeared":           opsAlert(p); break;
    case "ownership":             complianceReview(p); break;
    case "status":
      if (p.event_code.startsWith("INSOLVENCY_")) riskTeam(p);
      break;
    // legalRepresentatives, address, financial, other — same names as Topograph
  }
  res.status(200).end();
});

// Discover applicable sources, then create
const sources = await kausate.GET("/v2/monitors/sources", { params: { query: { kausateId } } });
await kausate.POST("/v2/monitors", {
  body: {
    kausateId: "co_de_2tT6getO1qTAvO3iaGnju6",
    sources: sources.data.sources.map(s => s.name),
    scheduleCron: "0 9 * * *",
    webhookUrl: "https://your-app.example.com/webhooks/kausate-monitors/<unguessable-segment>",
    categoriesFilter: ["status", "ownership", "disappeared"],
    autoDeactivateCategories: ["disappeared"],
  },
});
```

Shifts: strip Svix; idempotent on `event_id` not `monitor_id`; **N webhooks for N changes** (not one with `changeCategories: [...]`); discover sources first; use `categoriesFilter` to reduce noise.

---

## 18. Reference index

- `../auth-versioning.md` — header rename, `Kausate-Version` pinning, generated-client recipes
- `../endpoints.md` — full endpoint inventory and body shapes
- `../async-webhooks.md` — order lifecycle, receiver checklist, polling fallback
- `../status-and-dedup.md` — six order statuses, same-day dedup tuple, `bypassCache`
- `../customer-correlation.md` — `customerReference` vs `customerId` (`X-Customer-Id` header)
- `../documents.md` — two-step flow, pre-signed URLs, expiry handling
- `../monitors.md` — sources, categories, `event_code` taxonomy, monitor webhook payload, auto-deactivation
- `../identifiers.md` — native registry ID formats per jurisdiction
- `../errors-and-anti-patterns.md` — HTTP codes, common mistakes
- `../production-checklist.md` — pre-ship review

If anything here conflicts with `https://api.kausate.com/openapi.json`, the spec wins.
