# Errors

Every error response has a `detail` field. Most domain errors use a human-readable string:

```json
{ "detail": "Human-readable error description" }
```

Schema validation (`422`) uses a list of field errors. Some domain validation errors
instead use a structured `detail` object with a `code` field — for example
`text_too_long` when sending overlong text, or `recipient_equals_sender` (`400`) when a
WhatsApp send is addressed to the channel's own number. The per-endpoint shapes are in
the [API Reference](openapi.yaml).

### HTTP status codes

| Code | Meaning | What to do |
|---|---|---|
| `200` | Success | — |
| `201` | Created | — |
| `400` | Bad request | Check the `detail` field |
| `401` | Unauthorized | Check your `X-API-Key` header |
| `402` | Payment required | Trial ended or subscription inactive — see [Subscriptions & Billing](subscriptions.md) |
| `404` | Not found | Resource doesn't exist or belongs to another account |
| `422` | Validation error | Check required fields, types, and field constraints in `detail` |
| `429` | Rate limit exceeded | Back off and retry after `Retry-After` — see [rate limits](capabilities.md#rate-limits) |
| `502` | Meta API error | Upstream failure. Check `detail`. Retry may help. |
| `503` | Temporarily overloaded | Transient load shedding. Retry after `Retry-After`. |

### Send error codes

The three send endpoints answer `200` even when the send fails — the outcome is
in `success`, `status`, and `error_code`. See
[Delivery and retries](sending-messages.md#delivery-and-retries).

`error_code` is Meta's error code, passed through unchanged. It is present only
when the failure came from Meta; a rejection by Fiwano itself uses an HTTP
status code from the table above instead. `error` always carries a
human-readable description, and for media sends a hint about the likely cause.

| `error_code` | Meaning | Retried by Fiwano | What to do |
|---|---|---|---|
| `10`, `200` | Permission denied for this action | no | Reconnect the channel |
| `100` | Invalid parameter — Meta reuses this for several unrelated causes, including **a file above the size cap** | no | Read `error` for the specific cause; check `media_url`, `media_type`, recipient format, and [file size](capabilities.md#outbound-media-size) |
| `190` | Access token expired or revoked | no | Reconnect the channel |
| `368` | Account temporarily blocked for policy violations | no | Resolve in Meta Business Manager |
| `803` | Object does not exist or is unavailable | no | Check the recipient identifier |
| `131008` | Required parameter missing | no | Fix the request payload |
| `131009` | Parameter value invalid for this channel | no | Fix the request payload |
| `131026` | Recipient is not reachable on this platform | no | Verify the recipient |
| `131047`, `131057` | 24h re-engagement window closed | no | Send an approved WhatsApp template — see [messaging windows](capabilities.md#messaging-windows-24h) |
| `131051` | Unsupported message type for this channel | no | Check [channel capabilities](capabilities.md#channel-capabilities) |
| `131052` | Meta could not download `media_url` | no | Verify the URL returns `200`, `Content-Type` matches `media_type`, and the signature has not expired |
| `131053` | Meta could not process the media | **yes** | Often transient; check format and size if it persists |
| `131056` | Pair rate limit between this sender and recipient | **yes** | Slow down messages to that recipient |

Codes outside this table are passed through as Meta returns them. Anything not
recognised as permanent is treated as transient and retried.

### Unverified send outcomes

Rarely, Meta accepts a send but its response never reaches Fiwano — a lost
connection or a timeout during the reply. The message may or may not have been
delivered, and Meta offers no way to ask afterwards.

Fiwano does **not** retry these: an automatic retry would risk delivering the
same message twice. The response is `success: false`, `status: "failed"`, with
an `error` that states the outcome is unverified and carries no `error_code`.

**Check the conversation before resending.** The same caution applies if your
own HTTP client times out — see
[Response time](sending-messages.md#response-time).
