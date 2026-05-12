---
name: kausate-integration
description: Integrate or migrate to the Kausate company-data API (KYB / business registries / shareholder graphs / UBO / documents). Investigate the user's codebase first, ask the few questions the codebase can't answer, then generate production-grade integration code — typed client from the live OpenAPI, async + webhooks, customerId / customerReference, polling fallback. Use this skill whenever the user asks to integrate Kausate, fetch company data, do KYB, look up companies in business registries, retrieve shareholder graphs / UBO / company reports / annual accounts, wire up Kausate webhooks, or migrate from Kyckr, Moody's (kompany / Orbis / Maxsight / Bureau van Dijk), Topograph, OpenCorporates, or any other KYB / company-data provider.
---

# Kausate integration

Kausate is a real-time KYB / company-data API. It pulls live data from official business registries across 50+ jurisdictions (DE Handelsregister, GB Companies House, FR INPI, NL KVK, …). Most data endpoints are **async** because government registries are slow and unreliable — a production integration has to handle this correctly.

This skill is **interactive and progressive**. The body you're reading now is the orchestrator; specifics live in `references/`. Don't dump everything into the user's editor. Investigate first, ask targeted questions, then load only the references that match the path.

If anything below disagrees with `https://api.kausate.com/openapi.json`, the OpenAPI spec wins — it's generated from the live code.

---

## 1. Discovery loop — inspect first, ask second

Before writing code or asking generic questions, **read the user's codebase** so the questions you do ask are contextual and the answers shape what you generate. The user is being told a tool can "set this up for me" — wasting their time on questions you could answer by reading three files erodes trust.

### 1a. Scan these signals, silently

Inspect whatever is available. Skip what isn't.

| Signal | Where to look | What it tells you |
| --- | --- | --- |
| Language / runtime | `package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `*.csproj` | Which client-generation recipe to use (§2 below) |
| HTTP client convention | imports in existing API client files; e.g. `axios`, `fetch`, `requests`, `httpx`, `net/http`, `OkHttp`, `HttpClient` | Match the style they already use; don't introduce a new dependency unless asked |
| Existing webhook receivers | search for `/webhook`, `webhook` directories, `stripe-signature`, `svix`, `paddle`, or framework routes (`apps/*/api/webhooks/`, `routes/webhooks/`) | Tells you whether they have HTTPS receivers, where the new Kausate one should live, and what auth header pattern they already use |
| Existing competitor integration | grep for `kyckr`, `kompany`, `orbis`, `bvd`, `bvdinfo`, `topograph`, `opencorporates`, `passfort`, `maxsight`, `dnb`, `dun.bradstreet`, `creditsafe` in source, env templates, infra configs | Triggers a **migration** path — load the matching `references/migrations/from-*.md` |
| Typed-client generator already in use | `orval`, `openapi-typescript`, `openapi-python-client`, `oapi-codegen`, `openapi-generator`, `Kiota` in `package.json` / Makefile / scripts | Reuse it instead of installing another generator |
| Test convention | `pytest`, `vitest`, `jest`, `go test`, `*_test.go`, `tests/` layout | Where to put the integration tests you'll write |
| Secret-loading convention | `.env`, `.env.example`, dotenv, AWS SSM, Vault, framework-specific config | Where `KAUSATE_API_KEY` should be wired in |
| Deployment target / network | `Dockerfile`, `serverless.yml`, `vercel.json`, `fly.toml`, infra config | Influences webhook reachability (lambda cold starts? always-on? internal-only network?) |

If you find a competitor integration, **say so out loud**: *"I see `src/lib/kyckr.ts` — looks like you're currently on Kyckr. I'll load the Kyckr → Kausate migration reference."*

### 1b. Ask only the questions the codebase couldn't answer

After the scan, ask the **smallest set of questions needed** to get to good code. Default to three, never exceed four. Compress: if signals make a question obvious, skip it.

The fixed questions to consider, in priority order:

1. **Use case mix.** "Which capabilities do you need? Pick all that apply: (a) search companies → fetch company report, (b) UBO / shareholders, (c) shareholder graph (multi-level ownership), (d) document retrieval (registered extracts, articles of association), (e) form prefill / autocomplete UX. *Most integrations are (a)+(b); add more only if you'll actually use them — each one is its own endpoint and webhook flow.*"

2. **Webhook reachability.** "Can your service receive inbound HTTPS webhooks from the public internet? (Yes / not yet, will start with polling / on a private network — webhooks impossible.)" — If the codebase already has Stripe/Paddle/Svix webhook receivers, **don't ask** — instead confirm: *"You already receive Stripe webhooks at `app/api/webhooks/stripe`. I'll put the Kausate one alongside at `app/api/webhooks/kausate`. OK?"*

   When you propose the file layout, propose **all of it up front**, not just the receiver: the typed client module, the order-completion receiver, and the background-task / queue module if the codebase uses Celery / BullMQ / Sidekiq / cron. Partial layouts force the user to ask follow-up questions and slow the migration.

3. **API key source.** "Where should I read the Kausate API key from at runtime?" — Keep the answer in context and mirror it everywhere the key is referenced; don't reopen the question later. If the codebase already commits to one secret store (an `.env` file, an existing `SecretsManagerClient` / `vault.read(...)` / `getSecret(...)` call), **don't ask** — mirror that exact pattern for `KAUSATE_API_KEY` and just confirm before generating code. If they don't have a key yet, point them at [kausate.com/signup](https://www.kausate.com/signup) → dashboard → API keys.

4. **Migration source** — only ask if §1a didn't surface one. "Are you replacing an existing KYB / company-data provider? (Kyckr / Moody's-kompany / Moody's-Orbis-or-Maxsight / Topograph / OpenCorporates / Creditsafe / D&B / custom-built / no — greenfield.)"

5. **Jurisdictions** — only ask if the codebase shows it matters (e.g. they have a `countries: ['GB']` config). "Which jurisdictions do you need on day one? Coverage is 50+; the registry behavior differs a bit per country. *If 'global / all', I won't optimize for any one of them.*"

**Power-user escape hatch.** If the user says any of: *"just give me the whole guide,"* *"dump the skill,"* *"I know what I'm doing,"* *"no questions,"* — skip §1 entirely and read every reference in `references/` (still excluding `migrations/` unless relevant). Some integrators want the wall of text.

### 1c. Route to references

Based on the scan + answers, read **only** the reference files you need. Don't pre-load everything — that's why we split this skill.

**Reference paths depend on how this skill was installed.** Use whichever resolves:

- **Claude Code plugin** (`/plugin install kausate-integration@kausate`): references are on disk at `references/...md` relative to this SKILL.md.
- **`AGENTS.md` curl** (`curl docs.kausate.com/agents/kausate.md`): references aren't on disk — fetch them as URLs from `https://docs.kausate.com/agents/references/<name>.md` (e.g. `https://docs.kausate.com/agents/references/async-webhooks.md`). Use your AI assistant's URL-fetch tool.

| If the user needs … | Local file (plugin) | URL (AGENTS.md) |
| --- | --- | --- |
| Auth, env vars, version pinning, generated client | `references/auth-versioning.md` | `https://docs.kausate.com/agents/references/auth-versioning.md` |
| The endpoint inventory + native-id → `kausateId` translation | `references/endpoints.md` | `https://docs.kausate.com/agents/references/endpoints.md` |
| Async lifecycle + webhook receiver pattern | `references/async-webhooks.md` (almost always) | `https://docs.kausate.com/agents/references/async-webhooks.md` |
| `customerReference` vs `customerId` semantics | `references/customer-correlation.md` | `https://docs.kausate.com/agents/references/customer-correlation.md` |
| All six order statuses + same-day dedup + `bypassCache` | `references/status-and-dedup.md` | `https://docs.kausate.com/agents/references/status-and-dedup.md` |
| Document retrieval (two-step flow, pre-signed URLs) | `references/documents.md` | `https://docs.kausate.com/agents/references/documents.md` |
| Per-jurisdiction native ID formats | `references/identifiers.md` | `https://docs.kausate.com/agents/references/identifiers.md` |
| Error codes + anti-patterns to avoid generating | `references/errors-and-anti-patterns.md` | `https://docs.kausate.com/agents/references/errors-and-anti-patterns.md` |
| Pre-ship review | `references/production-checklist.md` | `https://docs.kausate.com/agents/references/production-checklist.md` |
| **Migrating from Kyckr** | `references/migrations/from-kyckr.md` | `https://docs.kausate.com/agents/references/migrations/from-kyckr.md` |
| **Migrating from Moody's** (kompany / Orbis / Maxsight / BvD) | `references/migrations/from-moodys.md` | `https://docs.kausate.com/agents/references/migrations/from-moodys.md` |
| **Migrating from Topograph** | `references/migrations/from-topograph.md` | `https://docs.kausate.com/agents/references/migrations/from-topograph.md` |

Each reference file is self-contained and concise. Reading three of them is faster and cleaner than skimming a 750-line monolith.

**When you load a reference, cite its path in your reply.** "Per `references/migrations/from-kyckr.md` …" or "From the polling section of `references/async-webhooks.md` …". This lets the user open the file directly and saves them a follow-up question. The deep migration playbooks especially deserve named citations — they encode hours of doc-walking the user shouldn't have to redo.

---

## 2. Hard rules — apply on every path

These four hold regardless of language, use case, or migration source. Internalize them before generating code.

1. **Pin `Kausate-Version` on every request.** Kausate uses date-based versioning (Cadwyn). If you don't send the header, you get the org's default version and breaking changes can land under you. Pick a date from the OpenAPI `info` block — currently `2026-05-01` — and pin it both on the request and in your client config.

2. **API key in env vars only.** `X-API-Key: $KAUSATE_API_KEY`. Never commit it. Add `.env*` to `.gitignore`. Rotate via the dashboard if leaked. Keys are org-scoped — there is no separate sandbox; every call hits production registries and consumes credits.

3. **Async + webhooks is the production pattern.** Most data endpoints return `{orderId, status: "running"}` immediately and finish via webhook callback (or polling fallback). **Don't write code that only handles `status === "completed"`** — handle all six statuses (`running`, `completed`, `failed`, `canceled`, `terminated`, `timedOut`). Sync mode (`/sync` endpoints) has a hard 300-second timeout and is for prototypes, not production.

4. **The OpenAPI spec is canonical.** When in doubt about a request/response shape, fetch `https://api.kausate.com/openapi.json` (requires `X-API-Key`) and read it. Don't infer schemas — they change at version boundaries.

---

## 3. The minimal first integration

When the user says *"just get me a working integration,"* this is what to generate. Inline the relevant pieces — don't make them read three references for a hello-world.

```bash
# 1. Pull the OpenAPI spec and generate a typed client
curl -H "X-API-Key: $KAUSATE_API_KEY" \
  https://api.kausate.com/openapi.json -o openapi.json
# Language-specific generator → see references/auth-versioning.md

# 2. Subscribe a webhook BEFORE placing orders (no backfill)
curl -X POST https://api.kausate.com/v2/webhooks \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Order completions",
    "url": "https://your-app.example.com/webhooks/kausate",
    "customHeaders": { "Authorization": "Bearer <your-shared-secret>" }
  }'

# 3. Translate a native registry ID → kausateId (cache the mapping)
curl -X POST https://api.kausate.com/v2/companies/search/sync \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "Content-Type: application/json" \
  -d '{"companyNumber":"33255959","jurisdictionCode":"nl"}'
# → result.searchResults[0].kausateId

# 4. Place an async order — result will arrive on your webhook
curl -X POST https://api.kausate.com/v2/companies/report \
  -H "X-API-Key: $KAUSATE_API_KEY" \
  -H "Kausate-Version: 2026-05-01" \
  -H "X-Customer-Id: end-customer-789" \
  -H "Content-Type: application/json" \
  -d '{"kausateId":"co_nl_…","customerReference":"kyc-case-12345"}'
# → { "orderId": "ord_…", "status": "running" }
```

Receiver-side: 2xx in under 5 seconds, queue real work, be idempotent on `orderId`. Full receiver checklist in `references/async-webhooks.md`.

---

## 4. Reference index

- **Live OpenAPI spec** (auth required): `https://api.kausate.com/openapi.json`
- **Human-readable docs**: <https://docs.kausate.com>
- **Per-page markdown twins** (for AI tools): append `.md` to any docs URL — e.g. `https://docs.kausate.com/api-reference/async-sync.md`
- **Doc index for AI tools**: <https://docs.kausate.com/llms.txt>
- **Changelog**: <https://docs.kausate.com/changelog>
- **Canonical source for this skill**: <https://github.com/kausate/agents>

If anything in this skill conflicts with `openapi.json`, the spec wins.
