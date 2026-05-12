# `customerReference` vs `customerId` — use both, deliberately

Both are optional, both are URL-safe strings (`[a-zA-Z0-9_-]`), both capped at **150 characters**. They have distinct purposes and they're sent at different layers.

| Field | What it is | Typical value | Set per | How to send |
|---|---|---|---|---|
| `customerReference` | Per-request correlation id you generate | KYC case id, order id from your system | request | JSON body field |
| `customerId` | Persistent identifier for **your end customer** | the user / tenant / account in your product | end customer | `X-Customer-Id` header |

`customerId` is a header, not a body field. Sending it in the body fails Pydantic validation because the request models are `extra=forbid`. The header makes it easy for a thin gateway to inject a per-end-customer id without rewriting bodies.

Both round-trip: they appear in the immediate `ProductOrderResponse`, in the polling response, and in the webhook payload. Use them like this:

- **`customerReference`** — set per request. Lets you correlate a webhook delivery back to the originating call. Always set this in production.
- **`customerId`** — set per end customer. Lets you partition usage and cost in `/v2/analytics/breakdowns?groupBy=customerId`, and is the primary key your handler uses to route results to the right tenant / user.

## Anti-patterns

- **Putting `customerId` in the request body.** It's `X-Customer-Id` header. Body submission returns 422.
- **Sending `customerReference` longer than 150 chars or with non URL-safe characters.** Will 422.
- **Reusing the same `customerReference` across orders of the same `kausateId` on the same UTC day.** Same-day dedup will return the existing `orderId` instead of placing a new one — see `status-and-dedup.md`.
