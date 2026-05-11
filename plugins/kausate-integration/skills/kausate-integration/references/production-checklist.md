# Production checklist

Before you ship:

- [ ] API key in env vars, never in source.
- [ ] `Kausate-Version` pinned to a specific date on every request.
- [ ] Webhook subscribed *before* any orders go out.
- [ ] Webhook receiver authenticates via the custom headers you set at subscription time.
- [ ] Webhook receiver returns 2xx in < 5 s and queues real work.
- [ ] Idempotent on `orderId`.
- [ ] All six `status` values handled distinctly (`running`, `completed`, `failed`, `canceled`, `terminated`, `timedOut`).
- [ ] Polling is a fallback path only; webhook is the primary signal.
- [ ] Reconciliation job for orders still `running` past their expected window.
- [ ] `customerReference` set per request, `customerId` set per end customer (header).
- [ ] Retry on 5xx with exponential backoff; do not retry on 4xx.
- [ ] Log `orderId` on every request and every webhook delivery for tracing.
- [ ] Cost monitoring via `/v2/analytics/summary` or the dashboard — every call costs credits.
