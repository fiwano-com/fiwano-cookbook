# Capabilities

What each channel supports, the license tiers, and the platform limits.

### Channel Capabilities

All three channels are connected the same way (OAuth). The table below shows what each channel supports.

| Feature | WhatsApp | Instagram | Facebook Messenger |
|---|---|---|---|
| Outbound text — max length | 4096 chars | 1000 chars | 2000 chars |
| Outbound media (Pro) | image, audio, video, document | image, audio, video, document | image, audio, video, document |
| Template messages (Pro) | ✅ Required outside 24h window | ❌ Not supported | ❌ Not supported |
| Incoming webhooks — text | ✅ `type: "text"` | ✅ `type: "text"` | ✅ `type: "text"` |
| Incoming webhooks — media (Pro) | image, audio, video, document, sticker | image, audio, video, document | image, audio, video, document |
| Delivery statuses | `sent` `delivered` `read` `failed` | `delivered` `read` | `delivered` `read` |
| Recipient format | Phone number without `+`  | IGSID | PSID — |
| 24h window workaround | Use approved templates | None — wait for user to message | None — wait for user to message |
| Channel identifier | `phone_number_id` | `ig_account_id` | `page_id` |
| Sender profile | `data.from_name` (from Meta contacts) | Via [profile endpoint](webhooks.md#sender-profile) | Via [profile endpoint](webhooks.md#sender-profile) |

> **Note:** Each Meta account (phone number, Instagram account, or Facebook Page) can only be connected to one Fiwano user at a time.

### License Tiers

Fiwano offers two license tiers. Each connected channel requires an active license.

| Tier | Monthly | Capabilities |
|---|---|---|
| **Starter** | $12 | Unlimited inbound and outbound text messages, delivery statuses |
| **Pro** | $19 | Everything in Starter **+** inbound media with files, outbound media via HTTPS URL (signed URLs supported), WhatsApp template management and sending |

New accounts start with a 7-day free trial (Pro tier). For the billing lifecycle and how a channel's subscription state is reported, see [Subscriptions & Billing](subscriptions.md). For how this flat fee relates to Meta's own per-message charges, see [Messaging Costs Explained](https://fiwano.com/documentation/messaging-costs).

### Rate limits

Each API key is allowed **~600 requests/minute (sustained)** with short bursts
above that. Exceeding it returns HTTP `429` with a `Retry-After` header telling
you how many seconds to wait; successful responses carry `X-RateLimit-Limit` and
`X-RateLimit-Remaining` so you can pace yourself. Under exceptional aggregate
load the platform may briefly shed requests with HTTP `503` + `Retry-After` —
treat both `429` and `503` the same way: honor `Retry-After` and retry. This is a
generous guardrail, not a hard product cap — if you need more sustained
throughput, [contact us](mailto:contact@fiwano.com). Meta has its own per-channel
limits (shown in Meta Business Manager, not controlled by Fiwano).

### Messaging windows (24h)

Meta restricts when you can message a user outside an open conversation:

- **WhatsApp** — you can send regular text only within **24 hours** of the
  customer's last message. Outside the window, use an approved template via
  `POST /api/v1/messages/send-template`. This is a Meta policy.
- **Instagram & Facebook Messenger** — you can reply only within **24 hours** of
  the user's last message. There is no template workaround — wait for the user to
  message again.

### Media limits

- **Inbound media** (images, audio, video, documents) is stored temporarily for
  **60 minutes**. Download it via `GET /api/v1/media/{media_id}` promptly after
  the webhook; files are cleaned up automatically after expiry. Maximum file
  size: **10 MB**.
- **Pro license required** for sending/receiving media and using WhatsApp
  templates. With a Starter license, inbound media arrives as
  `type: "unsupported"` with `upgrade_required: "pro"`. See
  [Subscriptions & Billing](subscriptions.md).
