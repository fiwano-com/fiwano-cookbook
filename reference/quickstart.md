# Quickstart

Fiwano puts WhatsApp, Instagram and Messenger behind one REST API. This is the
core loop in **four steps** — authenticate, connect a channel, receive a message,
reply — plus an optional fifth for messaging outside the 24-hour window. Every
request uses the base URL `https://fiwano.com` and carries your key in the
`X-API-Key` header.

### 1. Get and verify your API key

Every new account gets a **7-day free trial with full functionality** — every
channel type, media and templates, no card required. Open **API Keys** in the
[portal](https://fiwano.com) and create a key: the full key is shown **once**,
starts with `mip_live_`, and is stored only as a hash — lost keys can't be
recovered, so revoke and recreate if needed. Keep it in an environment variable
(e.g. `FIWANO_API_KEY`); never hardcode or commit it.

**Verify the key works** by listing channels:

```bash
curl https://fiwano.com/api/v1/channels -H "X-API-Key: $FIWANO_API_KEY"
```

A valid key on a fresh account (no channels yet) returns **`200`** with an empty
list — this is the success signal that you're authenticated and ready for step 2:

```json
{ "channels": [], "total": 0 }
```

An invalid or missing key returns `401`. Error shapes and status codes:
[Errors](errors.md).

### 2. Connect a channel

There are **two ways to connect** — pick the one that matches who owns the
account, both covered in [Channels](channels.md):

- **Your own channel** — connect it in the [portal](https://fiwano.com)
  (Channels → Connect), no code. Best when you operate the accounts yourself.
  The prerequisites (the asset must belong to a Meta Business Portfolio, and you
  must be its admin) are spelled out there.
- **Your end-users' channels** — an embedded OAuth flow you drive from your app:
  whitelist a `redirect_uri`, create a setup URL, the user completes Meta login
  inside it, and you exchange the returned one-time `code` for a `channel_id`.

OpenAPI operations for this step:

- Manage channels: `GET /api/v1/channels`, `GET /api/v1/channels/{channel_id}`, `PATCH /api/v1/channels/{channel_id}`, `DELETE /api/v1/channels/{channel_id}`
- Embedded connect flow: `POST /api/v1/channels/setup-url`, `POST /api/v1/channels/exchange-code`
- Redirect-URI whitelist: `GET /api/v1/redirects`, `POST /api/v1/redirects`, `DELETE /api/v1/redirects/{redirect_id}`

**Success:** you have a `channel_id` (the embedded flow returns it straight from
`exchange-code`), and `GET /api/v1/channels` now lists the channel with
`"is_active": true` — it can send and receive. That `channel_id` is what you pass
to every send and receive call from here on.

### 3. Receive a message

Replying to inbound is Fiwano's core use case, so set up receiving **before**
sending. Two parts:

**1. Enable events on the channel.** Delivery is opt-in — **by default no events
are delivered**. Set `webhook_events` (and a `webhook_url`) on the channel, and
enable only the events you actually handle (start with `message.received`). The
per-channel event list and how to configure it are in
[Channels](channels.md).

**2. Handle the webhook.** Fiwano POSTs each enabled event to your `webhook_url`.
Your endpoint **must verify the `X-Webhook-Signature`** (HMAC-SHA256 with the
channel's `webhook_secret`) and **respond HTTP 2xx within ~5 seconds** —
otherwise Fiwano retries with backoff and emails you. Payload shapes and the
signature check are in [Receiving Messages](webhooks.md).

An inbound `message.received` carries the two identifiers you need to reply
(marked below):

```jsonc
{
  "event": "message.received",
  "channel_id": "a1b2c3d4e5f67890",   // ← which of your channels received it
  "channel_type": "whatsapp",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "message_id": "wamid.xxx",
    "from": "1234567890",              // ← who sent it — reply to this
    "from_name": "John Doe",
    "type": "text",
    "text": "Hello!"
  }
}
```

- **`channel_id`** (top level) — the channel the message arrived on.
- **`data.from`** — the sender's id: phone number (WhatsApp), IGSID (Instagram),
  or PSID (Facebook). This is exactly what you pass back as `recipient`.

From a handler you'll often also call:

- `PATCH /api/v1/channels/{channel_id}` — set or update `webhook_events` / `webhook_url`
- `GET /api/v1/media/{media_id}` — download received media
- `GET /api/v1/channels/{channel_id}/profile/{user_id}` — look up the sender's profile

**Success:** message your connected channel from a real device; your endpoint
receives a `message.received` webhook with a valid signature and returns 2xx.
You're now receiving.

### 4. Reply to it

With receiving in place, send outbound. The everyday case is a **free-form reply
within the 24-hour window** after a user messages you — plain text or media, no
approval needed. This is the core move: answer the sender by feeding the **same**
identifiers straight back — the webhook's `channel_id` as `channel_id`, and
`data.from` as `recipient`:

```bash
curl -X POST https://fiwano.com/api/v1/messages/send \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"channel_id": "a1b2c3d4e5f67890", "recipient": "1234567890", "text": "Thanks for your message!"}'
```

- Send: `POST /api/v1/messages/send` (text), `POST /api/v1/messages/send-media` (media)

**Success:** the call returns an accepted message with a `message_id`; if you
enabled delivery events in step 3, you'll then receive `message.sent` /
`message.delivered` webhooks tracking it. Read
[Sending Messages](sending-messages.md) for media and more.

That covers the core loop — connect, receive, reply. Step 5 is optional.

### 5. WhatsApp templates — messaging outside the 24-hour window (optional)

Free-form messages only reach a user **inside** the 24-hour window. To start a
conversation, or to reply after the window has closed, WhatsApp requires a
pre-approved **template** (WhatsApp only). Skip this step if you only ever reply
within the window — see the 24-hour window in
[Capabilities](capabilities.md#messaging-windows-24h).

Read [WhatsApp Templates](templates.md) for the create/review
lifecycle, then send the approved template.

- Send a template: `POST /api/v1/messages/send-template`
- Manage templates: `GET /api/v1/channels/{channel_id}/templates`, `POST /api/v1/channels/{channel_id}/templates`, `GET|PUT|DELETE /api/v1/channels/{channel_id}/templates/{template_id}`

### Next steps

- **No code?** Use the verified [n8n node](n8n.md) — same channels,
  same events, drag-and-drop.
- **Media and templates** — [Sending Messages](sending-messages.md).
- **Limits, windows and tiers** — [Capabilities](capabilities.md).
- **The full machine-readable contract** — [API Reference](openapi.yaml).
</content>
</invoke>
