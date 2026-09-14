# Capabilities

What each channel supports, the license tiers, and the platform limits.

### Channel Capabilities

All three channels are connected the same way (OAuth). The table below shows what each channel supports.

| Feature | WhatsApp | Instagram | Facebook Messenger |
|---|---|---|---|
| Outbound text — max length | 4096 chars | 1000 chars | 2000 chars |
| Outbound media (Pro) | image, audio, video, document, sticker (WebP file) | image, audio, video, document | image, audio, video, document, sticker (Meta catalog `sticker_id`) |
| Template messages (Pro) | ✅ Required outside 24h window | ❌ Not supported | ❌ Not supported |
| Incoming webhooks — text | ✅ `type: "text"` | ✅ `type: "text"` | ✅ `type: "text"` |
| Incoming webhooks — media (Pro) | image, audio, video, document (stickers as `image` + `media.sticker`) | image, audio, video, document | image, audio, video, document (stickers as `image` + `media.sticker`) |
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

Message sends are limited to **10 accepted send attempts per second per channel**
across all API keys. The limit is shared by text, media, and template sends, so
creating another key does not increase one channel's allowance while one key can
drive many channels independently. Exceeding it returns HTTP `429` with
`Retry-After`. Other public API operations do not share a product-wide RPS cap.
During exceptional outbound saturation, a send can briefly return HTTP `503` +
`Retry-After`; honor the header and retry. Meta also enforces its own channel and
recipient limits (shown in Meta Business Manager, not controlled by Fiwano).

### Messaging windows (24h)

Meta restricts when you can message a user outside an open conversation:

- **WhatsApp** — you can send regular text only within **24 hours** of the
  customer's last message. Outside the window, use an approved template via
  `POST /api/v1/messages/send-template`. This is a Meta policy.
- **Instagram & Facebook Messenger** — you can reply only within **24 hours** of
  the user's last message. There is no template workaround — wait for the user to
  message again.

### Media limits

#### Outbound file size

Meta downloads your `media_url` and enforces its own per-platform caps. Fiwano
does not re-check the file, so an oversize file is rejected by Meta with
`error_code` `100` and the message is **not** retried — see
[Errors](errors.md#send-error-codes).

| Media type | WhatsApp | Instagram | Facebook Messenger |
|---|---|---|---|
| Image | 5 MB (JPEG, PNG) | 8 MB (JPEG, PNG) | 8 MB (JPEG, PNG, GIF) |
| Sticker | 100 KB static / 500 KB animated (WebP, 512×512 px) | not available | no file — sent by Meta catalog `sticker_id` |
| Video | 16 MB (MP4, 3GPP) | 25 MB (MP4, OGG, AVI, MOV, WebM) | 25 MB |
| Audio | 16 MB (AAC, AMR, MP3, MP4, OGG) | 25 MB (AAC, M4A, WAV, MP4) | 25 MB |
| Document | 100 MB (PDF, Office, text) | 25 MB (PDF) | 25 MB |

Facebook Messenger caps video, audio, and documents at 25 MB, but **images at
8 MB** (the stricter limit Meta applies to URL-based uploads, which is how Fiwano
sends). Meta does not enumerate accepted formats per type; handle the oversize or
unsupported-format rejection rather than relying on a fixed list.

These are Meta's limits and Meta may change them. Note that encoding overhead
can push a file over the cap even when its size on disk looks safe.

#### Inbound file size

- **Inbound media** (images, audio, video, documents) is stored temporarily for
  **60 minutes**. Download it via `GET /api/v1/media/{media_id}` promptly after
  the webhook; files are cleaned up automatically after expiry. Maximum file
  size: **10 MB**.
- **Pro license required** for sending/receiving media and using WhatsApp
  templates. With a Starter license, inbound media arrives as
  `type: "unsupported"` with `upgrade_required: "pro"`. See
  [Subscriptions & Billing](subscriptions.md).
