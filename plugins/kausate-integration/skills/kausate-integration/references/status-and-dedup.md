# Order status values and same-day dedup

## Order status — handle all six

Async runs finish in one of six states. Treat anything other than `completed` as not-success.

| Status | Meaning | Action |
|---|---|---|
| `running` | Still processing | Wait for webhook or keep polling |
| `completed` | Success — `result` is populated | Use it |
| `failed` | Permanent failure — `error` populated | Surface to the user; retry with `bypassCache: true` if appropriate (registry was down, etc.) |
| `canceled` | Order canceled and acknowledged | Treat as not-found from your side |
| `terminated` | Forcefully terminated (rare) | Same as `failed` |
| `timedOut` | Reached its time limit (workflow-level 7 days; can hit registry-side limits earlier) | Retry — registries can be slow; the next attempt may succeed |

Don't write code that only checks `status === "completed"`. Make the non-completed branch explicit.

## Same-day dedup and `bypassCache`

If you place the same request twice on the same UTC calendar day (same `kausateId`, same workflow, same `customerReference`, same `customerId`), Kausate returns the existing `orderId` instead of starting a new workflow. This protects against duplicate billing on retries.

If you genuinely need a fresh registry call (e.g. data was missing, registry was down on the first try), set `bypassCache: true` on the request body. This skips the dedup key, starts a new workflow, and consumes new credits.

`bypassCache` does **not** bypass any internal result cache — only the same-day dedup. The result you get back is always live from the registry.
