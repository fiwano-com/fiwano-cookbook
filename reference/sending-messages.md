# Sending Messages

Fiwano has three send endpoints — plain text, media, and WhatsApp templates. All
take a `channel_id` and a `recipient`. The `recipient` format depends on the
channel (phone number for WhatsApp, IGSID for Instagram, PSID for Facebook) — see
the recipient row in [Capabilities](capabilities.md#channel-capabilities). Full
request/response schemas are in the [API Reference](openapi.yaml); this page
is the task guide.

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
own chunking and ordering. See [Capabilities](capabilities.md).

### Media messages

`POST /api/v1/messages/send-media` — **Pro license required.** Meta fetches the file
directly from `media_url`; Fiwano never downloads or stores it. Pass `media_type`
(`image`, `audio`, `video`, `document`) and an HTTPS `media_url`.

**Use a signed URL for non-public content** — S3/GCS/R2 presigned, Azure SAS, or an
HMAC-signed URL on your own server, with expiry ≥ 5 min. A public URL is reachable
by anyone who learns it.

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-media \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "recipient": "1234567890",
    "media_type": "image",
    "media_url": "https://my-bucket.s3.amazonaws.com/photo.jpg?X-Amz-Signature=...&X-Amz-Expires=600",
    "caption": "Your order photo"
  }'
```

Always check `success`. On failure the response includes a Meta `error_code` —
e.g. `131052` (Meta couldn't download the URL) or `131053` (unsupported
format/size, or Meta rate-limited your host's network — try a major cloud
provider). File-size caps and the full error-code table live in
[Capabilities](capabilities.md) and the [API Reference](openapi.yaml).

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

### Delivery and retries

`send` and `send-media` return `200` with a `status` field, because what happens
after Meta accepts the request matters:

- **`sent`** — Meta accepted it. Track the rest via delivery-status webhooks
  (`message.delivered` / `read` / `failed`) — see
  [Receiving Messages](webhooks.md#delivery-status-tracking).
- **`queued`** — a transient Meta failure (network, 5xx, rate limit). Fiwano
  retries in the background (up to 7 times over ~20 min). You get an early-warning
  email after 3 failed retries and a final email if they're exhausted.
- **`failed`** (`success: false`) — a permanent error (bad recipient, oversize
  text, malformed payload). **Not retried** — fix the request and resend. The
  channel owner is emailed.

So `200` does not by itself mean "delivered" — always read `success` and `status`.
