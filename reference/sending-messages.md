# Sending Messages

Fiwano has three send endpoints — plain text, media, and WhatsApp templates. All
take a `channel_id` and a `recipient`. The `recipient` format depends on the
channel (phone number for WhatsApp, IGSID for Instagram, PSID for Facebook) — see
the recipient row in [Capabilities](capabilities.md#channel-capabilities). Full
request/response schemas are in the [API Reference](openapi.yaml); this page
is the task guide.

### Send WhatsApp messages with the API

For a normal WhatsApp reply inside the 24-hour customer service window, use the
plain text endpoint below with a WhatsApp `channel_id` and the recipient's phone
number. You authenticate with your Fiwano `X-API-Key`; you do not need a separate
Meta or WhatsApp API key in your application.

Outside the 24-hour window, WhatsApp requires an approved template message. That
uses `/api/v1/messages/send-template` and is described in
[Template messages](#template-messages). Instagram DM and Facebook Messenger use
the same text endpoint for ordinary replies, with IGSID or PSID as `recipient`.

### Text messages

`POST /api/v1/messages/send` — works on all channel types.

```bash
curl -X POST https://fiwano.com/api/v1/messages/send \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"channel_id": "a1b2c3d4e5f67890", "recipient": "1234567890", "text": "Hello! Your order is ready."}'
```

The response carries a `message_id` (a Fiwano UUID) that every later delivery-status
webhook references. Text has a per-platform length cap (WhatsApp 4096, Facebook
2000, Instagram 1000) — oversize text is rejected with `400 text_too_long` before
Meta is called. Fiwano does **not** auto-split; split on your side to preserve your
own chunking and ordering. Empty or whitespace-only `text` is rejected with `422`.

An obviously invalid `recipient` (empty, no digits, or a non-numeric PSID/IGSID) is
rejected with `400 invalid_recipient` before Meta is called; any other WhatsApp number
is passed to Meta as is, and Meta's own rejection comes back as `status: "failed"`.
See [HTTP errors](errors.md#http-errors).

### Media messages

`POST /api/v1/messages/send-media` — **Pro license required.** Meta fetches the file
directly from `media_url`. Pass `media_type`
(`image`, `audio`, `video`, `document`, or `sticker` — see [Stickers](#stickers))
and an HTTPS `media_url`.

**Use a signed URL for non-public content** — S3/GCS/R2 presigned, Azure SAS, or an
HMAC-signed URL on your own server, with expiry ≥ 20 min so background retries can
still fetch it. A public URL is reachable
by anyone who learns it.

**Keep files small and hosting fast.** Meta downloads the file while your request
is waiting, so a compressed image on a fast host is accepted in a couple of
seconds, while a large file on a slow host can take Meta a minute or more. Smaller
files mean faster, more predictable delivery and fewer sends that finish in the
background — see [Response time](#response-time).

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-media \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "recipient": "1234567890",
    "media_type": "image",
    "media_url": "https://my-bucket.s3.amazonaws.com/photo.jpg?X-Amz-Signature=...&X-Amz-Expires=1800",
    "caption": "Your order photo"
  }'
```

Always check `success` and `status`. Permanent failures include a Meta
`error_code` — for example `131052` when Meta cannot download the URL; temporary
ones, such as `131053`, return `queued` and are retried like text. File-size caps are in
[Capabilities](capabilities.md#outbound-media-size) and the full
error-code table is in [Errors](errors.md#send-error-codes).

#### Stickers

`media_type: "sticker"` sends a sticker; what it takes depends on the channel,
because Meta's platforms differ:

| Channel | Field | What Meta accepts |
|---|---|---|
| WhatsApp | `media_url` | a WebP file, 512×512 px, up to 100 KB (static) or 500 KB (animated) |
| Facebook Messenger | `sticker_id` | a sticker from Meta's own catalog — `369239263222822` is the thumbs up — or the `media.sticker_id` of a sticker a user sent you |
| Instagram | — | no sticker message exists; send an `image` instead |

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-media \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"channel_id": "a1b2c3d4e5f67890", "recipient": "1234567890",
       "media_type": "sticker", "media_url": "https://my-bucket.s3.amazonaws.com/thanks.webp?X-Amz-Signature=..."}'
```

`caption` and `filename` are ignored for stickers. The wrong field for the
channel — `sticker_id` on WhatsApp, `media_url` on Messenger, any sticker on
Instagram — is answered with `400 invalid_media_request` before Meta is called
(`reason` says which). A WebP that breaks WhatsApp's rules, or a WebP sent as
`image`, fails at once with Meta code `131053` and a hint; it is not retried.
An inbound sticker arrives as `image` with `media.sticker: true` — see
[Receiving Messages](webhooks.md#media-messages).

### Response time

`POST /messages/send-media` is **synchronous and can be slow**. Fiwano never
downloads your file: we hand Meta the `media_url` and **Meta fetches it inside
your request**. The wait is therefore proportional to the file size and to how
fast your own hosting serves it. A 12-second call for a large file is normal.
Fiwano waits for Meta for up to **30 seconds**; if Meta has not answered by then,
the call returns `queued` and Fiwano completes the send in the background — see
[Delivery and retries](#delivery-and-retries).

Text and template sends are not affected — they carry no file and typically
complete in well under a second.

**Set your HTTP client timeout to at least 35 seconds** for `send-media`. Some
environments cap this for you and cannot wait that long — AWS API Gateway stops
at 29 seconds, and serverless functions often default to 10–15 seconds.

> **If your client times out, the message may still have been sent.** Meta can
> accept it after you stopped waiting. Resending then delivers it twice.
> Retry only after confirming the message is absent from the conversation.

To keep media sends fast, serve `media_url` from storage close to your users
(S3/GCS/R2 with a CDN) and keep files well under the
[size caps](capabilities.md#outbound-media-size).

### Template messages

`POST /api/v1/messages/send-template` — **WhatsApp only, Pro required.** Use a
pre-approved template to start a conversation outside the 24-hour window (see
[Capabilities](capabilities.md#messaging-windows-24h)). Only `APPROVED`
templates can be sent — to create and manage them, see
[WhatsApp Templates](templates.md).

Provide variable values keyed by component. **Positional** templates (`{{1}}`,
`{{2}}`) take arrays:

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-template \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "template_name": "order_confirmation",
    "language": "en_US",
    "recipient": "1234567890",
    "variables": {
      "header": ["Summer Sale"],
      "body": ["Pablo", "ORD-123", "25%"],
      "buttons": [{"index": 0, "value": "promo25"}]
    }
  }'
```

**Named** templates (`{{customer_name}}`) take objects:

```bash
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "template_name": "welcome_message",
    "language": "en_US",
    "recipient": "1234567890",
    "variables": {"body": {"customer_name": "Pablo", "order_number": "ORD-123"}}
  }'
```

Omit `variables` entirely if the template has none.

Template sends return the same response shape as text and media sends. The
`message_id` is a Fiwano UUID; keep it to correlate later delivery webhooks:

```json
{
  "success": true,
  "message_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "error": null,
  "error_code": null,
  "status": "sent"
}
```

### Delivery and retries

All three send endpoints answer `200` with `success`, `message_id`, `status`,
`error` and `error_code`. A `200` alone does not mean the message went out —
read `status`:

| `status` | `success` | Meaning | What to do |
|---|---|---|---|
| `sent` | `true` | Meta accepted the message | Follow it with [delivery-status webhooks](webhooks.md#delivery-status-tracking) |
| `queued` | `true` | Fiwano is finishing the send in the background; `message_id` is final | Do not resend. The outcome arrives as webhooks: the usual statuses, or [`message.failed`](webhooks.md#event-types) |
| `failed` | `false` | Not sent, and not retried | Read `error` and `error_code` — see [send error codes](errors.md#send-error-codes) |

A request Fiwano refuses before calling Meta — `text_too_long`,
`invalid_recipient`, an inactive channel, a rate limit — gets an
[HTTP error](errors.md#http-errors) instead of `failed`. On `send`
and `send-media`, a `failed` message is also reported to the channel owner by
email.

In the portal, each channel's **Delivery problems** page (Messages tab) lists
the last 7 days of messages that are `queued` for a retry (with the attempt
number and the next try) or `failed`, with Meta's code and text — the same
`error` the API and `message.failed` return. This includes WhatsApp messages
that Meta accepted and later reported as failed.

A send is `queued` for one of two reasons:

- **Meta failed temporarily** (text and media) — a network error, a Meta `5xx`,
  a rate limit, or a temporary error code. Fiwano retries up to 7 times over
  ~20 minutes. The channel owner is emailed after 3 failed retries, and again if
  all of them fail; you then receive `message.failed`. `send-template` does not
  retry: any error Meta returns is `failed`, and you decide whether and when to
  resend.
- **Meta answered slowly** (any send) — see below.

<a id="unverified-send-outcomes"></a>

#### Slow Meta responses

Meta occasionally takes more than 30 seconds — sometimes over a minute — to
answer a send, most often while it downloads a large file. The call then returns
`queued` and Fiwano waits for Meta to confirm the message:

- Usually the confirmation comes and the normal `message.sent` /
  `message.delivered` / `message.read` webhooks follow.
- If Meta confirms nothing within a few minutes, Fiwano sends a text or media
  message once more (never a template). The first attempt may have gone through
  after all, so in rare cases the recipient gets the message twice — a duplicate
  is preferred to a lost message.
- If nothing is confirmed in the end, the message becomes `failed`: you receive
  `message.failed` and the channel owner is emailed.

Do not resend on your side while a message is `queued`. If your own HTTP client
times out before the answer, see [Response time](#response-time).
