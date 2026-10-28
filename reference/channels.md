# Channels

A **channel** is one connected Meta asset — a WhatsApp number, an Instagram
account, or a Facebook Page — that you send and receive messages through. This
page covers how to connect, manage and reconnect channels. Field-level schemas
are in the **[API Reference](openapi.yaml)**.

### Prerequisites

Before connecting any channel — WhatsApp, Instagram or Facebook Messenger — make
sure the conditions below are met. They apply equally to the Portal flow and the
API flow; if the first two are missing, Meta stops the OAuth popup before a
channel can be created.

- **The asset belongs to a Meta Business Portfolio** (Business Manager). The
  "asset" is the WhatsApp number's WABA, the Facebook Page, or — for Instagram —
  a Business or Creator Instagram account linked to a Facebook Page that is owned
  by a Business Portfolio.
- **The Facebook user signing in has full admin rights** on that Business
  Portfolio and on the asset itself. A user without admin role sees the relevant
  choice in the popup greyed out.
- **Fiwano is the app in control of conversations** (Instagram and Facebook
  Messenger). Meta gives one app control of each conversation at a time
  (*Conversation Routing*), so Fiwano must be the *Default routing app* to reply.
  Set it in Facebook Page → Settings → Page setup → *Instagram conversation
  routing* / *Messenger conversation routing* (for Instagram without a Page: Meta
  Business Suite → Settings → Integrations → *Conversation Routing*), turn off
  *Take control of conversations* for other apps or disconnect them, and don't
  work these chats from the Meta Business Suite / Page inbox or with Meta AI —
  either hands control to Meta's own inbox. If Fiwano is not in control, incoming
  messages still reach your webhook but replies are rejected with
  [error `10`](errors.md#error-10).

### Option A: Via Portal (self-service)

Use this to connect **your own** channels, no code required.

1. Go to **Channels → Connect Channel** in the portal.
2. Select the channel type (WhatsApp, Instagram, or Facebook Messenger).
3. Complete the Meta OAuth flow in the popup window.
4. Configure the **Webhook URL** and select **Webhook Events** in channel settings.
5. By default, no events are enabled — select which events to forward to your endpoint.
6. Generate a **webhook secret** so deliveries are signed — see [Webhook secret](#webhook-secret).

The webhook URL must be an absolute HTTPS URL reachable from Fiwano. Explicit
ports from 1 to 65535 are supported; embedded credentials and URL fragments are not.

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

Redirect URI patterns must use HTTPS and cannot target localhost or a loopback
address. Explicit ports from 1 to 65535 are supported, including non-standard
HTTPS ports such as `https://yourapp.com:4426/callback`. Exact URIs are safest
and recommended. When a wildcard is necessary, it may appear in the path/query
or as one complete left-most hostname label (`*.example.com`), but cannot replace
the whole hostname, part of a label, or the port. Embedded credentials, URL
fragments, and the reserved query keys `code`, `status`, `channel_type`, and
`error` are rejected.

To associate a setup flow with your authenticated tenant or administrator,
generate a high-entropy, single-use opaque nonce, store it server-side with that
context, and put only the nonce in the redirect URI. Register a narrowly scoped
pattern such as `https://yourapp.com/callback?state=*`, then request the setup URL
with `https://yourapp.com/callback?state=BASE64URL_NONCE`. Fiwano preserves
`state` and appends its own parameters, for example
`?state=BASE64URL_NONCE&code=...&status=success&channel_type=whatsapp`. Use a
URL-safe value and do not place tenant/user identifiers or other sensitive data
directly in the URI.

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

The same endpoint also reconnects channels; there is no separate reconnect API.
If the user connects a Meta account that is already one of your channels,
`exchange-code` returns that existing `channel_id` — see
[Reconnecting a channel](#reconnecting-an-inactive-channel).

The request is refused with `402` when the account has no active subscription,
and with `409` when every subscription slot for that channel type is already
taken and none of the occupying channels can be reconnected through this flow.
The `409` body is structured: `detail.code` is `no_free_slot` and
`detail.occupied_by` lists the channels holding the slots — see
[subscription slots](#subscription-slots) for how to free one.

**Step 3 — User completes Meta OAuth.** After approval, the user is redirected to
your `redirect_uri` with a one-time `code` parameter:

```
https://yourapp.com/callback?code=abc123...
```

On failure, the redirect instead carries two query params — branch your logic on
`error` only:

| Query param | How to use it |
|---|---|
| `error` | Machine-readable code. **Branch on this.** `access_denied` — the user cancelled or did not complete the Meta dialog. `slot_occupied` — the user connected a *different* Meta account than the one holding your subscription slot; `message` names the channel to reconnect or release (see [subscription slots](#subscription-slots)). `session_expired` — the setup URL expired before the flow finished; request a new one. `setup_failed` — anything else that stopped setup (e.g. no Instagram Business account was accessible with the permissions granted). |
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

Most events exist on every channel; `message.sent` is WhatsApp-only and
`conversation.referral` is Instagram and Facebook only. The full list with
payloads is on the **[Receiving Messages](webhooks.md#event-types)** page.
An event you list that isn't valid for the channel type is ignored, not an error.

One event has an extra knob: `message.echo` (copies of messages your business sends
outside Fiwano) delivers just the message by default. Set the channel's boolean
`echo_statuses` field — in the connect call, via `PATCH /api/v1/channels/{id}`, or in
the Portal — to also receive delivered/read statuses for echoed messages through your
regular status events. Details: **[message.echo](webhooks.md#message-echo)**.

**Your endpoint owns the other half of this contract.** Once events are enabled, Fiwano
POSTs each one to your `webhook_url`, and your endpoint **must respond with HTTP 2xx
within 5 seconds**; otherwise Fiwano retries and emails you — see
**[Retry Policy](webhooks.md#retry-policy)**. So enable **only the events
you actually handle**, and return 2xx as soon as you've accepted the payload (do
slower work afterwards).

### Webhook secret

The `webhook_secret` is the HMAC key Fiwano uses to **sign webhook deliveries**, so
your endpoint can confirm a request genuinely came from Fiwano and was not altered in
transit. When a channel has a secret, every delivery carries an
`X-Webhook-Signature: sha256=<hmac>` header — see **[Verifying Signatures](webhooks.md#verifying-signatures)**
for the verification snippet. A channel with no secret receives **unsigned** deliveries.

How a secret first appears differs by how you connect — and this is the one place the
Portal and the API deliberately behave differently:

- **Portal (Option A):** a new channel has **no secret**, and saving a webhook URL
  does not create one. Set it yourself in channel settings: click **Generate random**
  for a random 64-character secret, or type your own and **Save** (16–64
  characters). The value is revealed **once**, immediately after.
- **API (Option B):** when you set `webhook_url` and the channel has no secret yet,
  Fiwano **auto-generates** one (64-character hex) and returns it in the
  `exchange-code` / `PATCH /api/v1/channels/{id}` response — so channels you connect by
  API are **signed by default**. To use a specific value instead, pass your own
  `webhook_secret` (**at most 64 characters**) in that same call.

**Reading it back.** The value is only returned the moment it is set or changed — in
the Portal's one-time reveal, or in the response of the `exchange-code` or
`PATCH /api/v1/channels/{id}` call that set or changed it. Every other response of
those two calls (only events changed, webhook removed, a reconnect that keeps the
existing secret) carries `webhook_secret: null`. `GET /api/v1/channels` and
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
- Reconnecting a channel **keeps** its existing secret (see
  [Reconnecting a channel](#reconnecting-an-inactive-channel) below).
- Treat it like a password: store it in a secret manager, never commit it, and
  verify signatures using a constant-time comparison (as in the verification snippet).

### Managing channels

| Task | Endpoint |
|---|---|
| List all channels (active and inactive), each with its current subscription state | `GET /api/v1/channels` |
| Inspect one channel | `GET /api/v1/channels/{id}` |
| Update webhook URL / secret / events, or the subscription binding | `PATCH /api/v1/channels/{id}` |
| Deactivate a channel | `DELETE /api/v1/channels/{id}` |

Each channel carries a `subscription` block describing its billing state — see
**[Subscriptions & Billing](subscriptions.md)** for what the
combinations mean. Full field lists live in the **[API Reference](openapi.yaml)**.

**Deactivation is a soft delete.** `DELETE` stops the channel from sending and
receiving, but does not erase it — its `channel_id` and history are preserved so
you can reconnect later. The channel also remains owned by the same Fiwano
account: deactivation does not release its WhatsApp number, Instagram account or
Facebook Page for connection to another Fiwano account. If the channel must move
between accounts, contact `support@fiwano.com`.

### Subscription slots

Each subscription grants **one slot per channel type** — one WhatsApp, one
Instagram, one Facebook. A slot stays occupied while a channel is bound to it,
**including a deactivated channel**: the binding is what lets you reconnect that
channel later without buying another subscription.

`GET /api/v1/subscriptions` shows which channel sits in each slot and how many
slots are free; each channel reports its own `subscription.id` in return.

Send `subscription_id` to `PATCH /api/v1/channels/{channel_id}` to change that. A
subscription ID moves the channel there — no downtime, and it does not have to be
deactivated first, but a move to a Starter subscription stops media and template
sending immediately. An empty string releases the slot, and that is allowed only
for a channel already deactivated with `DELETE /api/v1/channels/{channel_id}`, so
a slot is never freed as a side effect of a settings update.

**Releasing a slot is effectively permanent.** Once another channel takes the
freed slot, the released one can no longer be reconnected until a slot is free
again. It is not erased and its Meta identity stays owned by your Fiwano account —
but treat the release as retiring that channel, not pausing it.

Replacing a channel when you have a single subscription:

```text
GET    /api/v1/subscriptions           → find the subscription and its occupied slot
DELETE /api/v1/channels/{old_id}       → deactivate the channel you are replacing
PATCH  /api/v1/channels/{old_id}       → {"subscription_id": ""} frees the slot
POST   /api/v1/channels/setup-url      → user connects the new Meta account
POST   /api/v1/channels/exchange-code  → new channel takes the free slot
```

### Channel health

A channel can stay active and still be unable to work, when Meta no longer lets
Fiwano work with the account or number: the app was removed in Meta Business
settings, a required permission was revoked, the Page or WhatsApp account became
unavailable, the WhatsApp number is not on the WhatsApp Business Platform (the
*WhatsApp Business App* connection was not completed), Meta blocked the
account, or Meta banned the WhatsApp number. Fiwano then marks the channel **Action required** in the portal and
emails the account owner; the notice and the email say what to do. Until the
cause is fixed, sends may fail — for example with [error `190`](errors.md#send-error-codes).

Every channel in the API carries the same state as `health`:

```json
"health": {
  "status": "action_required",
  "reason": "(#190) Error validating access token: …",
  "since": "2026-10-06T10:00:00Z"
}
```

`status` is `ok` or `action_required` — branch on it. `reason` is human-readable
text, usually Meta's own error with its code, which you can search in Meta's
documentation; do not branch on it. `since` is when the problem was detected.
`health` returns to `ok` by itself once Meta allows access again, and on
reconnect. `is_active` stays `true`: it turns `false` only when the channel is
deactivated. Failures of individual messages are not part of `health` — they
come in the send response and the `message.failed` webhook, and are listed on
the channel's **Delivery problems** page in the portal.

<a id="reconnecting-an-inactive-channel"></a>

### Reconnecting a channel

Reconnect a deactivated channel, or one marked *Action required*, by running the
**same connection flow again for the same Meta account** (same WhatsApp number,
Instagram account, or Facebook Page). When Meta has blocked the account or banned
the number, resolve it in Meta first — reconnecting alone does not clear it:

- The existing channel is **updated in place** — its `channel_id`, webhook
  URL/secret/events and history are preserved. No new channel is created and your
  stored `channel_id` mapping stays valid.
- Reconnecting requires an **active subscription**: the channel must still hold
  one, or you must have a free slot. Otherwise `setup-url` is refused (`402` or
  `409`, see above) — bind a subscription first, in the portal's Billing page or
  via `PATCH /api/v1/channels/{channel_id}`.
- If the user completes the dialog for a **different** Meta account while the
  deactivated channel still holds the slot, the flow is refused at the end with
  `error=slot_occupied` and nothing is created. Reconnect the same account, or
  release the slot first.
- A Meta account owned by a different Fiwano account cannot be connected, even
  when that channel is inactive. If it is your channel, contact
  `support@fiwano.com` to request an ownership release.
