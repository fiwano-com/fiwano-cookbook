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
own chunking and ordering. The `text` value must contain at least one non-whitespace
character; empty or whitespace-only values are rejected with `422` before Meta is
called. Leading and trailing whitespace in otherwise valid text is preserved. See
[Capabilities](capabilities.md).

### Media messages

`POST /api/v1/messages/send-media` — **Pro license required.** Meta fetches the file
directly from `media_url`; Fiwano never downloads or stores it. Pass `media_type`
(`image`, `audio`, `video`, `document`) and an HTTPS `media_url`.

**Use a signed URL for non-public content** — S3/GCS/R2 presigned, Azure SAS, or an
HMAC-signed URL on your own server, with expiry ≥ 20 min so background retries can
still fetch it. A public URL is reachable
by anyone who learns it.

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
`error_code` — for example `131052` when Meta cannot download the URL. Failures
that can recover, including `131053` processing/fetcher failures and `131056`
pair rate limits, return `queued` and use the same durable retry schedule as text.
File-size caps are in
[Capabilities](capabilities.md#outbound-media-size) and the full
error-code table is in [Errors](errors.md#send-error-codes).

### Response time

`POST /messages/send-media` is **synchronous and can be slow**. Fiwano never
downloads your file: we hand Meta the `media_url` and **Meta fetches it inside
your request**. The wait is therefore proportional to the file size and to how
fast your own hosting serves it. A 12-second call for a large file is normal;
Fiwano gives up on Meta after **30 seconds**.

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

All three send endpoints return `200` with the common `success`, `message_id`,
`error`, `error_code`, and `status` fields, because what happens after Meta
accepts the request matters:

- **`sent`** — Meta accepted it. Track the rest via delivery-status webhooks
  (`message.delivered` / `read` / `failed`) — see
  [Receiving Messages](webhooks.md#delivery-status-tracking).
- **`queued`** — a transient Meta failure (network, 5xx, rate limit). Fiwano
  retries in the background (up to 7 times over ~20 min). You get an early-warning
  email after 3 failed retries and a final email if they're exhausted. Only
  `send` and `send-media` can return `queued`.
- **`failed`** (`success: false`) — the request will not be retried. For `send`
  and `send-media`, this means a permanent error (bad recipient, oversize text
  or media, malformed payload), and the channel owner is emailed. `send-template`
  does not retry automatically, so any Meta send error is returned as `failed`;
  the caller can decide whether and when to resend.
  `failed` is also used for the rare case where Meta's response was lost and the
  outcome cannot be confirmed — see
  [unverified send outcomes](errors.md#unverified-send-outcomes).

So `200` does not by itself mean "delivered" — always read `success` and `status`.
