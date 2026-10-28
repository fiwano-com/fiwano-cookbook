# Sending Messages

Fiwano has four send endpoints — plain text, media, interactive messages
(buttons, cards, quick replies, lists), and WhatsApp templates. All
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
[WhatsApp Templates](templates.md#sending). Instagram DM and Facebook Messenger use
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

`caption` (up to 1024 characters) and `filename` (up to 255) are WhatsApp fields:
`caption` is shown under an `image`, `video` or `document`, and `filename` names a
`document`. Anywhere else they are accepted and ignored — a WhatsApp `audio` or
sticker has no caption, and on Instagram and Messenger an attachment carries
neither: the file arrives on its own, named after its URL. To add text there, send
it with [`/messages/send`](#text-messages) before or after the file.

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
An inbound sticker arrives as `image` — with `media.sticker: true` on WhatsApp and
Messenger, as a plain GIF on Instagram — see
[Receiving Messages](webhooks.md#media-messages).

### Response time

`POST /messages/send-media` is **synchronous and can be slow**. Fiwano never
downloads your file: we hand Meta the `media_url` and **Meta fetches it inside
your request**. The wait is therefore proportional to the file size and to how
fast your own hosting serves it. A 12-second call for a large file is normal.
Fiwano waits for Meta for up to **30 seconds**; if Meta has not answered by then,
the call returns `queued` and Fiwano completes the send in the background — see
[Delivery and retries](#delivery-and-retries).

Interactive messages with `cards` behave the same way, because Meta downloads
every card image. Text, template and other interactive sends carry no file and
typically complete in well under a second.

**Set your HTTP client timeout to at least 35 seconds** for `send-media` and for
`send-interactive` with cards. Some
environments cap this for you and cannot wait that long — AWS API Gateway stops
at 29 seconds, and serverless functions often default to 10–15 seconds.

> **If your client times out, the message may still have been sent.** Meta can
> accept it after you stopped waiting. Resending then delivers it twice.
> Retry only after confirming the message is absent from the conversation.

To keep media sends fast, serve `media_url` from storage close to your users
(S3/GCS/R2 with a CDN) and keep files well under the
[size caps](capabilities.md#outbound-media-size).

### Interactive messages

`POST /api/v1/messages/send-interactive` — **Pro license required.** A message the
user answers with a tap: reply buttons, a link button, product-style cards,
quick-reply chips or a WhatsApp list. One request shape works on every channel;
each channel receives its own native message. Like text, interactive messages are
sent inside the [24-hour window](capabilities.md#messaging-windows-24h)
— outside it on WhatsApp, use a [template](#template-messages) with quick-reply
buttons.

Send `text` plus exactly one of:

| Field | What the user sees | WhatsApp | Instagram / Messenger |
|---|---|---|---|
| `buttons` (1–3) | Text with buttons under it | ✓ reply buttons, or one url button | ✓ any mix of reply and url |
| `cards` (1–10) | Picture, title, subtitle and buttons; 2+ cards scroll sideways | ✓ | ✓ |
| `quick_replies` (1–13) | Text with chips that disappear after a tap | — | ✓ |
| `list` (up to 10 rows) | Text with a button that opens a list of options | ✓ | — |

A button is `{"type": "reply", "id": "...", "title": "..."}` — tapping it sends
the `title` back to you — or `{"type": "url", "title": "...", "url": "https://..."}`,
which opens the link and sends nothing. `title` is up to 20 characters, `id` up to
256 and unique within the message.

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-interactive \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "recipient": "1234567890",
    "text": "Which size do you need?",
    "buttons": [
      {"type": "reply", "id": "size:S", "title": "S"},
      {"type": "reply", "id": "size:M", "title": "M"},
      {"type": "reply", "id": "size:L", "title": "L"}
    ]
  }'
```

Cards — each one needs an HTTPS `image_url` (JPEG or PNG), a `title` (up to 80
characters, one line), an optional `subtitle` (up to 80) and 1–3 buttons:

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-interactive \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "channel_id": "b2c3d4e5f6789012",
    "recipient": "17841400000000000",
    "cards": [
      {"image_url": "https://cdn.example.com/rain-jacket.jpg",
       "title": "Rain Jacket", "subtitle": "$89 · sizes S–XL",
       "buttons": [
         {"type": "reply", "id": "add_to_cart:SKU-1", "title": "Add to cart"},
         {"type": "url", "title": "View on site", "url": "https://shop.example.com/p/1"}
       ]},
      {"image_url": "https://cdn.example.com/fleece.jpg",
       "title": "Fleece", "subtitle": "$59 · sizes S–L",
       "buttons": [
         {"type": "reply", "id": "add_to_cart:SKU-2", "title": "Add to cart"},
         {"type": "url", "title": "View on site", "url": "https://shop.example.com/p/2"}
       ]}
    ]
  }'
```

Quick replies (Instagram, Messenger) and a list (WhatsApp):

```json
{"channel_id": "c3d4e5f678901234", "recipient": "25550000000000000",
 "text": "Pick a colour", "quick_replies": [
   {"id": "colour:red", "title": "Red"}, {"id": "colour:blue", "title": "Blue"}]}
```

```json
{"channel_id": "a1b2c3d4e5f67890", "recipient": "1234567890",
 "text": "How should we deliver?", "list": {"button": "Options", "sections": [
   {"title": "Delivery", "rows": [
     {"id": "ship:express", "title": "Express", "description": "Tomorrow, $9"},
     {"id": "ship:standard", "title": "Standard", "description": "3–5 days, free"}]}]}}
```

When the user taps a reply button, a chip or a list row, you receive a
[`message.received`](webhooks.md#button-taps) of `type: "text"` whose
`text` is the label and `button_id` is the `id` you set. Typed answers and taps go
through the same handler.

**What differs by channel.** These are Meta's rules; Fiwano checks them before
calling Meta and answers `400 invalid_interactive_request` with a `reason` and a
`hint`, instead of sending something other than you asked for:

| Rule | WhatsApp | Instagram / Messenger | `reason` |
|---|---|---|---|
| Buttons | 1–3 reply buttons **or** exactly one url button | Up to 3, any mix | `whatsapp_button_mix` |
| `text` with `cards` | Required for 2+ cards (shown above them); with one card it is shown under the card's title | Not accepted — send it first with `/messages/send` | `text_required`, `text_with_cards_unsupported` |
| Card buttons | The same on every card (one url button each, or the same number of reply buttons) | Free per card | `card_buttons_mismatch` |
| Card title + subtitle | Up to 160 characters together, sent as a bold title line and a subtitle line | 80 + 80 | `text_too_long` |
| `text` length | 1024 (`buttons`, `cards`), 4096 (`list`) | 640 (`buttons`), 1000 Instagram / 2000 Messenger (`quick_replies`) | `text_too_long` |
| `quick_replies` | Not available — use `buttons` or `list` | ✓ | `quick_replies_unsupported` |
| `list` | ✓ | Not available — use `quick_replies` or `buttons` | `list_unsupported` |

Instagram shows cards and quick replies only in the mobile app, not on
instagram.com. Meta downloads card images while your request waits, as with
[media](#response-time): keep them small and fast to serve. Delivery, `queued`
retries and statuses work exactly as for media — see
[Delivery and retries](#delivery-and-retries); a card image Meta cannot download
fails at once with `131052`.

### Template messages

`POST /api/v1/messages/send-template` — **WhatsApp only, Pro required.** Sends an
approved template, the way to start a conversation outside the 24-hour window (see
[Capabilities](capabilities.md#messaging-windows-24h)). Variables, media
headers and authentication codes are described in
[WhatsApp Templates → Sending a template](templates.md#sending). The
response and the delivery rules are the same as for text and media — see below.

### Delivery and retries

All send endpoints answer `200` with `success`, `message_id`, `status`,
`error` and `error_code`. A `200` alone does not mean the message went out —
read `status`:

| `status` | `success` | Meaning | What to do |
|---|---|---|---|
| `sent` | `true` | Meta accepted the message | Follow it with [delivery-status webhooks](webhooks.md#delivery-status-tracking) |
| `queued` | `true` | Fiwano is finishing the send in the background; `message_id` is final | Do not resend. The outcome arrives as webhooks: the usual statuses, or [`message.failed`](webhooks.md#event-types) |
| `failed` | `false` | Not sent, and not retried | Read `error` and `error_code` — see [send error codes](errors.md#send-error-codes) |

A request Fiwano refuses before calling Meta — `text_too_long`,
`invalid_recipient`, an inactive channel, a rate limit — gets an
[HTTP error](errors.md#http-errors) instead of `failed`. On `send`,
`send-media` and `send-interactive`, a `failed` message is also reported to the
channel owner by email.

In the portal, each channel's **Delivery problems** page (Messages tab) lists
the last 7 days of messages that are `queued` for a retry (with the attempt
number and the next try) or `failed`, with Meta's code and text — the same
`error` the API and `message.failed` return. This includes WhatsApp messages
that Meta accepted and later reported as failed.

A send is `queued` for one of two reasons:

- **Meta failed temporarily** (text, media and interactive) — a network error, a Meta `5xx`,
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
- If Meta confirms nothing within a few minutes, Fiwano sends a text, media or
  interactive message once more (never a template). The first attempt may have gone through
  after all, so in rare cases the recipient gets the message twice — a duplicate
  is preferred to a lost message.
- If nothing is confirmed in the end, the message becomes `failed`: you receive
  `message.failed` and the channel owner is emailed.

Do not resend on your side while a message is `queued`. If your own HTTP client
times out before the answer, see [Response time](#response-time).
