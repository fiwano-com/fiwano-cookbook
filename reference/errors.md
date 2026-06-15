# Errors

Every error response uses the same shape — a `detail` field with a human-readable description:

```json
{ "detail": "Human-readable error description" }
```

Some validation errors carry a structured `detail` object instead (for example `text_too_long` when sending overlong text); the per-endpoint shapes are in the [API Reference](openapi.yaml).

### HTTP status codes

| Code | Meaning | What to do |
|---|---|---|
| `200` | Success | — |
| `201` | Created | — |
| `400` | Bad request | Check the `detail` field |
| `401` | Unauthorized | Check your `X-API-Key` header |
| `402` | Payment required | Trial ended or subscription inactive — see [Subscriptions & Billing](subscriptions.md) |
| `404` | Not found | Resource doesn't exist or belongs to another account |
| `429` | Rate limit exceeded | Back off and retry after `Retry-After` — see [rate limits](capabilities.md#rate-limits) |
| `502` | Meta API error | Upstream failure. Check `detail`. Retry may help. |
| `503` | Temporarily overloaded | Transient load shedding. Retry after `Retry-After`. |
