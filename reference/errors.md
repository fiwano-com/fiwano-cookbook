# Errors

Every error response has a `detail` field. Most domain errors use a human-readable string:

```json
{ "detail": "Human-readable error description" }
```

Schema validation (`422`) uses a list of field errors. Some domain validation errors
instead use a structured `detail` object with a `code` field — for example
`text_too_long` when sending overlong text, `invalid_recipient` (`400`) when `recipient`
is empty, contains no digits, or is not a numeric PSID/IGSID on a Messenger/Instagram
channel (`reason` says which; `hint` says what to send instead), or
`recipient_equals_sender` (`400`) when a WhatsApp send is addressed to the channel's own
number. The per-endpoint shapes are in the [API Reference](openapi.yaml).

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
| `10`, `200` | Meta denies this action for the account | no | Not a token problem: the channel stays connected and keeps receiving. Read `error` (Meta's own text) and check the account in Meta Business Settings |
| `10` with *another app is controlling this thread* | Instagram/Messenger: another connected app owns the conversation | no | Make Fiwano the default routing app or disconnect the other app — see [Prerequisites](channels.md#prerequisites) |
| `100` | Invalid parameter — Meta reuses this for several unrelated causes, including **a file above the size cap** | no | Read `error` for the specific cause; check `media_url`, `media_type`, recipient format, and [file size](capabilities.md#outbound-media-size) |
| `190` | Access token expired or revoked | no | Reconnect the channel |
| `368` | Account temporarily blocked for policy violations | no | Resolve in Meta Business Manager |
| `551` | Messenger/Instagram: this person cannot be messaged right now — they blocked the Page, closed the chat, restricted business messages, or never messaged the Page | no | Nothing on your side; only the person can lift it. Do not resend automatically |
| `803` | Object does not exist or is unavailable | no | Check the recipient identifier |
| `131008` | Required parameter missing | no | Fix the request payload |
| `131009` | Parameter value invalid for this channel | no | Fix the request payload |
| `131026` | Recipient is not reachable on this platform | no | Verify the recipient |
| `131047` | 24h re-engagement window closed | no | Send an approved WhatsApp template — see [messaging windows](capabilities.md#messaging-windows-24h) |
| `131051` | Unsupported message type for this channel | no | Check [channel capabilities](capabilities.md#channel-capabilities) |
| `131052` | Meta could not download `media_url` | no | Verify the URL returns `200`, `Content-Type` matches `media_type`, and the signature has not expired |
| `131053` | Meta could not process the media | **yes** | Often transient; check format and size if it persists |
| `131056` | Pair rate limit between this sender and recipient | **yes** | Slow down messages to that recipient |
| `131057` | WhatsApp Business Account in maintenance mode (e.g. a throughput upgrade) | **yes** | Usually temporary; no action |

Codes outside this table are passed through as Meta returns them. Anything not
recognised as permanent is treated as transient and retried.

### Slow Meta responses

Occasionally Meta takes longer than 30 seconds — sometimes more than a minute —
to answer a send, most often while it downloads a large media file. Fiwano does
not fail the message: the call returns `success: true`, `status: "queued"`, and
the `message_id` is final. Fiwano then finishes the send with Meta on your
behalf. What you see:

- The usual `message.sent` / `message.delivered` / `message.read` webhooks for
  that `message_id`, exactly as for a message that returned `sent` right away.
- In rare cases the message reaches the recipient twice: if Meta has not
  confirmed the send within a few minutes, Fiwano sends it once more, and the
  first attempt may have gone through after all. A duplicate is preferred to a
  lost message.
- If the message cannot be confirmed at all, it becomes `failed` and the
  channel owner receives the delivery digest email.

**Do not resend on your side while the message is `queued`.** The same applies
if your own HTTP client times out — see
[Response time](sending-messages.md#response-time).
