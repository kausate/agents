# Documents — the two-step workflow

Document retrieval is two async orders, in this order:

## 1. List documents available for a company

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

`documentType` values include `annualAccounts`, `currentExtract`, `articlesOfAssociation`, `shareholderList`, `chronologicalExtract`, `officialFilings`, `registrationDetails`, `beneficialOwnersDetails`, `annualReturn`. `documentType` / `publicationDate` are sometimes `null`; filter accordingly.

## 2. Retrieve a specific document by `kausateDocumentId`

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

## Common patterns

- **Cache the listing for at least a day.** `documents/list` is itself an order; if your user is browsing the list, you don't want to re-list on every click. The `expiresAt` in the result indicates the index validity window.
- **Defer the actual fetch.** The list is cheap; the document fetch costs more. Only call `POST /v2/companies/documents` once the user clicks "download."
- **Don't store the `downloadLink`.** It expires. Persist the bytes you fetched, or persist `kausateDocumentId` and re-order if you need the file later.
