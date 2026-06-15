# Channels

A **channel** is one connected Meta asset — a WhatsApp number, an Instagram
account, or a Facebook Page — that you send and receive messages through. This
page covers how to connect, manage and reconnect channels.

For the exact request/response schema of every channel endpoint (fields, types,
status codes), see the **[API Reference](openapi.yaml)**. This page is the
task-level guide; it does not repeat the field tables. 

### Prerequisites

Before connecting any channel — WhatsApp, Instagram or Facebook Messenger — make
sure both conditions below are met. They apply equally to the Portal flow and the
API flow; if either is missing, Meta stops the OAuth popup before a channel can
be created.

- **The asset belongs to a Meta Business Portfolio** (Business Manager). The
  "asset" is the WhatsApp number's WABA, the Facebook Page, or — for Instagram —
  a Business or Creator Instagram account linked to a Facebook Page that is owned
  by a Business Portfolio.
- **The Facebook user signing in has full admin rights** on that Business
  Portfolio and on the asset itself. A user without admin role sees the relevant
  choice in the popup greyed out.

### Option A: Via Portal (self-service)

Use this to connect **your own** channels, no code required.

1. Go to **Channels → Connect Channel** in the portal.
2. Select the channel type (WhatsApp, Instagram, or Facebook Messenger).
3. Complete the Meta OAuth flow in the popup window.
4. Configure the **Webhook URL** and select **Webhook Events** in channel settings.
5. By default, no events are enabled — select which events to forward to your endpoint.

Saving a webhook URL in the Portal does **not** create a `webhook_secret`. Set one
explicitly so incoming deliveries are signed — see [Webhook secret](#webhook-secret).

### Option B: Via API (programmatic)

Use this when your application connects channels **on behalf of your end users**.

**Step 1 — Whitelist your redirect URI.** For security, the user can only be
redirected back to a URL you have pre-registered for your API key. Register the
URL(s) where users land after OAuth (wildcards are allowed, e.g.
`https://*.example.com/callback`):

```bash
curl -X POST https://fiwano.com/api/v1/redirects \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"uri_pattern": "https://yourapp.com/callback"}'
```

You manage these with `GET /api/v1/redirects` and `DELETE /api/v1/redirects/{id}`.

**Step 2 — Request a setup URL.** Pass one of your whitelisted redirect URIs. The
URL is valid until the `expires_at` returned in the response — open it in a
browser or popup for the user:

```bash
curl -X POST https://fiwano.com/api/v1/channels/setup-url \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"channel_type": "whatsapp", "redirect_uri": "https://yourapp.com/callback"}'
```

**Step 3 — User completes Meta OAuth.** After approval, the user is redirected to
your `redirect_uri` with a one-time `code` parameter:

```
https://yourapp.com/callback?code=abc123...
```

On failure, the redirect instead carries two query params — branch your logic on
`error` only:

| Query param | How to use it |
|---|---|
| `error` | Machine-readable code. **Branch on this.** `access_denied` — the user cancelled the Meta dialog. `setup_failed` — setup could not complete (e.g. no Instagram Business account was accessible with the permissions granted). |
| `message` | URL-encoded, human-readable English explanation, safe to display to the user. **Free-form and may change — never parse or branch on its text.** |

Example failure redirect:

```
https://yourapp.com/callback?error=setup_failed&message=We%20couldn%27t%20access%20any%20Instagram%20Business%20account...
```

**Step 4 — Exchange the code.** Within 5 minutes (single-use), exchange the code
for the channel. You can configure the webhook in the same call:

```bash
curl -X POST https://fiwano.com/api/v1/channels/exchange-code \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "abc123...",
    "webhook_url": "https://yourapp.com/webhooks/meta",
    "webhook_events": ["message.received", "message.delivered", "message.failed"]
  }'
```

The response returns your `channel_id` (store it — every other call uses it). When
you set `webhook_url` and pass no `webhook_secret`, Fiwano **auto-generates** one and
returns it here. It is returned **only in this response** — `GET` never shows it
again — so store it to verify webhook signatures ([Webhook secret](#webhook-secret)).
All fields except `code` are optional and can be set later via
`PATCH /api/v1/channels/{id}`.

### Webhook events

Webhook delivery is **opt-in per channel**: by default **no events are delivered**. You
choose what you receive by setting `webhook_events` — in the connect call, in the Portal,
or later via `PATCH /api/v1/channels/{id}`. Until you do, your endpoint gets nothing.

The available events depend on the channel type — WhatsApp exposes more
(`message.sent`, `message.failed`) than Instagram and Facebook. The full list with
payloads is on the **[Webhooks](webhooks.md#event-types)** page. An event you
list that isn't valid for the channel type is simply ignored, not an error.

**Your endpoint owns the other half of this contract.** Once events are enabled, Fiwano
POSTs each one to your `webhook_url`, and your endpoint **must respond with HTTP 2xx
within ~5 seconds**. A non-2xx response or a timeout counts as a failed delivery: Fiwano
**retries with backoff and emails you** — a warning after the 3rd failed attempt and an
alert when retries are exhausted. So enable **only the events you actually handle**, and
return 2xx as soon as you've accepted the payload (do slower work afterwards). Incoming
messages are saved on our side even if relay fails. Full delivery and retry behavior:
**[Webhooks → Retry Policy](webhooks.md#retry-policy)**.

### Webhook secret

The `webhook_secret` is the HMAC key Fiwano uses to **sign webhook deliveries**, so
your endpoint can confirm a request genuinely came from Fiwano and was not altered in
transit. When a channel has a secret, every delivery carries an
`X-Webhook-Signature: sha256=<hmac>` header — see **[Webhooks](webhooks.md)**
for the verification snippet. A channel with no secret receives **unsigned** deliveries.

How a secret first appears differs by how you connect — and this is the one place the
Portal and the API deliberately behave differently:

- **Portal (Option A):** a new channel has **no secret**, and saving a webhook URL
  does not create one. Set it yourself in channel settings: click **Generate random**
  for a random 64-character secret, or type your own and **Save** (minimum 16
  characters). The value is revealed **once**, immediately after.
- **API (Option B):** when you set `webhook_url` and the channel has no secret yet,
  Fiwano **auto-generates** one (64-character hex) and returns it in the
  `exchange-code` / `PATCH /api/v1/channels/{id}` response — so channels you connect by
  API are **signed by default**. To use a specific value instead, pass your own
  `webhook_secret` in that same call.

**Reading it back.** The value is only returned the moment it is set or changed — in
the Portal's one-time reveal, or in the `exchange-code` and
`PATCH /api/v1/channels/{id}` responses. `GET /api/v1/channels` and
`GET /api/v1/channels/{id}` never return it; they only report
`has_webhook_secret: true | false`. **Store the value when it is shown** — if you
lose it, your only option is to set a new one.

**Rotating it.** Set a new secret any time by passing a new `webhook_secret` to
`PATCH /api/v1/channels/{id}`, or with the Portal's **Generate random** / **Save**
actions. Updating only `webhook_url`/`webhook_events` leaves the secret untouched. A
change takes effect on the **very next delivery** — there is no overlap window, so
switch your verifier to the new secret at the same moment, or signatures will mismatch.

**Constraints and recommendations.**

- Use a high-entropy random string of **16–64 characters** (the Portal enforces the
  16-character minimum; the field stores up to 64). Auto-generated secrets are
  64-character hex — prefer those unless you have a reason to bring your own.
- The secret is **per channel** — each channel has its own, independent of the rest.
- Reconnecting an inactive channel **keeps** its existing secret (see
  [Reconnecting an inactive channel](#reconnecting-an-inactive-channel) below).
- Treat it like a password: store it in a secret manager, never commit it, and
  verify signatures using a constant-time comparison (as in the Webhooks example).

### Managing channels

| Task | Endpoint |
|---|---|
| List all channels (active and inactive), each with its current subscription state | `GET /api/v1/channels` |
| Inspect one channel | `GET /api/v1/channels/{id}` |
| Update webhook URL / secret / events | `PATCH /api/v1/channels/{id}` |
| Deactivate a channel | `DELETE /api/v1/channels/{id}` |

Each channel carries a `subscription` block describing its billing state — see
**[Subscriptions & Billing](subscriptions.md)** for what the
combinations mean. Full field lists live in the **[API Reference](openapi.yaml)**.

**Deactivation is a soft delete.** `DELETE` stops the channel from sending and
receiving, but does not erase it — its `channel_id` and history are preserved so
you can reconnect later. Fiwano also unsubscribes the channel's Meta webhook
resource only when it is safe to: a WABA subscription is kept if another active
WhatsApp channel uses the same WABA, and a Page subscription is kept if another
active Instagram/Facebook channel uses the same Page.

### Reconnecting an inactive channel

A channel goes inactive when it is deactivated (`DELETE /api/v1/channels/{id}`) or
when its Meta connection can no longer be maintained (for example, the account
owner revoked access in Meta). To bring it back, run the **same connection flow
again for the same Meta account** (same WhatsApp number, Instagram account, or
Facebook Page):

- The existing channel is **reactivated in place** — its `channel_id`, webhook
  URL/secret/events and history are preserved. No new channel is created and your
  stored `channel_id` mapping stays valid.
- Reconnecting requires an **active license**: the channel must still hold one, or
  you must have a free license slot. Otherwise the flow is refused — attach a
  license in Billing first.
- A Meta account that is currently active under a different Fiwano account cannot
  be reconnected (`"already connected to another account"`).
