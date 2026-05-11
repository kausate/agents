# Auth, version pinning, and generated client

## Authentication

All requests use an API key sent via the `X-API-Key` header:

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" https://api.kausate.com/v2/...
```

- Get an API key from [kausate.com/signup](https://www.kausate.com/signup) → dashboard → API keys.
- **Read from environment variables only.** Never commit keys. Add `.env*` to `.gitignore`.
- Rotate via the dashboard if a key leaks.
- API keys are org-scoped. Auth is `X-API-Key` only — no OAuth, no JWT for the public API.
- **No sandbox or test environment.** Every call hits production registries and consumes credits. Use small test budgets during development.

## Pin the API version with `Kausate-Version`

Kausate uses date-based API versioning (Cadwyn). Pin a version on every request via the `Kausate-Version` header so request/response shapes don't drift under you:

```
Kausate-Version: 2026-05-01
```

If you don't pin, you get the org's default version (or the latest if no default is set), and breaking changes can land during a major-release window. **Always pin in production.** Read the `info` block of `openapi.json` for the version list.

> **Important:** Different API versions expose different paths. In `2025-04-01` the report endpoint was `POST /v2/companies/{kausateId}/report`; in `2026-05-01` it's `POST /v2/companies/report` with `kausateId` in the body. Always match your path style to the version you've pinned. Generating the client from `openapi.json` while sending the matching `Kausate-Version` header keeps these in sync automatically.

## Generate a typed client from `openapi.json`

Kausate publishes its OpenAPI spec at `https://api.kausate.com/openapi.json`. The endpoint requires the `X-API-Key` header (it isn't anonymous):

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" \
  https://api.kausate.com/openapi.json \
  -o openapi.json
```

Then generate a typed client. Pick the matching ecosystem — and if the codebase already uses a generator (Orval / openapi-typescript / oapi-codegen / etc.), reuse it instead of installing another.

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

## Anti-patterns

- **Hardcoding the API base URL across environments.** Use `KAUSATE_API_BASE_URL` so dev / staging / prod can be swapped via env.
- **Storing the API key in source.** Read from env. Never commit `.env`.
- **Skipping `Kausate-Version`.** You will get bitten when the latest version moves on.
- **Mixing path styles for different `Kausate-Version`s.** With `2026-05-01` use `POST /v2/companies/report` (`kausateId` in body). With `2025-04-01` use `POST /v2/companies/{kausateId}/report`. Don't mix.
