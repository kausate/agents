# Auth, version pinning, and generated client

## Authentication

All requests use an API key sent via the `X-API-Key` header:

```bash
curl -H "X-API-Key: $KAUSATE_API_KEY" https://api.kausate.com/v2/...
```

- Get an API key from [kausate.com/signup](https://www.kausate.com/signup) → dashboard → API keys.
- **Read from environment variables or a managed secret store — never commit keys.** Add `.env*` to `.gitignore`.
- Rotate via the dashboard if a key leaks.
- API keys are org-scoped. Auth is `X-API-Key` only — no OAuth, no JWT for the public API.
- **No sandbox or test environment.** Every call hits production registries and consumes credits. Use small test budgets during development.

### Loading the API key — common secret stores

Mirror whatever the rest of the codebase uses. Common patterns:

**Plain `.env` / dotenv** (Node, Python, Go, most local dev):
```bash
# .env (gitignored)
KAUSATE_API_KEY=sk_live_...
```
```ts
const key = process.env.KAUSATE_API_KEY!;
```
```python
import os
key = os.environ["KAUSATE_API_KEY"]
```

**Pydantic Settings** (Python / FastAPI):
```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr

class Settings(BaseSettings):
    kausate_api_key: SecretStr
    class Config: env_file = ".env"

key = Settings().kausate_api_key.get_secret_value()
```

**AWS Secrets Manager** (Lambda, ECS, EKS):
```python
import boto3, json
sm = boto3.client("secretsmanager")
secret = json.loads(sm.get_secret_value(SecretId="kausate/api-key")["SecretString"])
key = secret["KAUSATE_API_KEY"]
```
```ts
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";
const sm = new SecretsManagerClient({});
const { SecretString } = await sm.send(new GetSecretValueCommand({ SecretId: "kausate/api-key" }));
const key = JSON.parse(SecretString!).KAUSATE_API_KEY;
```

**AWS SSM Parameter Store** (lighter alternative to Secrets Manager):
```python
import boto3
ssm = boto3.client("ssm")
key = ssm.get_parameter(Name="/kausate/api_key", WithDecryption=True)["Parameter"]["Value"]
```

**GCP Secret Manager**:
```python
from google.cloud import secretmanager
client = secretmanager.SecretManagerServiceClient()
key = client.access_secret_version(name="projects/<id>/secrets/kausate-api-key/versions/latest").payload.data.decode()
```

**Azure Key Vault**:
```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
client = SecretClient(vault_url="https://<vault>.vault.azure.net/", credential=DefaultAzureCredential())
key = client.get_secret("kausate-api-key").value
```

**HashiCorp Vault**:
```python
import hvac
client = hvac.Client(url="https://vault.internal", token=os.environ["VAULT_TOKEN"])
key = client.secrets.kv.v2.read_secret_version(path="kausate")["data"]["data"]["api_key"]
```

**Doppler** (drop-in `.env` replacement):
```bash
doppler run -- node server.js   # secrets injected as env vars
# then in code: process.env.KAUSATE_API_KEY
```

**1Password Connect** (server-side secrets):
```ts
import { OnePasswordConnect } from "@1password/connect";
const op = OnePasswordConnect({ serverURL, token });
const item = await op.getItemByTitle("vault-id", "Kausate API key");
const key = item.fields!.find(f => f.label === "credential")!.value!;
```

**Don't fall back to plain `.env` for production deploys** when the codebase already commits to a secret store — that's the path that leaks keys to dev laptops and source control. Match the existing pattern; if no pattern exists yet, prefer the platform-managed store the team already uses for other secrets.

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
