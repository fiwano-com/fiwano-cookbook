# Fiwano Integration Playbook

A step-by-step guide to integrating Fiwano — WhatsApp, Instagram and Messenger through one API — in **four clear steps**: authenticate, connect a channel, receive messages, and reply.

Each step explains what to do, points to the relevant file in [`reference/`](reference/), and names the OpenAPI operations it uses. The prose explains *how*; [`reference/openapi.yaml`](reference/openapi.yaml) is the exact, machine-readable contract (every field, type and status code). Follow it yourself — or hand it to an AI coding agent (see below).

> Base URL is `https://fiwano.com`. Never commit real API keys, tokens or webhook secrets.

## Using this with an AI coding agent

This cookbook is also built to hand straight to an AI agent (Cursor, Claude Code, etc.). You keep working in **your own project** — the cookbook is just the agent's reference:

1. Copy this cookbook into your project (for example as a `fiwano/` folder), or clone it next to your code and give the agent access to it.
2. Point the agent at this file:
   > "Use `fiwano/PLAYBOOK.md` to integrate Fiwano. Follow the steps and open the referenced files as needed."

The agent then reads only what each step references, instead of loading everything at once.

---

## Step 1 — Authenticate and verify your key

Every request carries your API key in the `X-API-Key` header. Create a key on the **API Keys** page in the [portal](https://fiwano.com): the full key is shown **once**, starts with `mip_live_`, and is stored only as a hash — lost keys can't be recovered, so revoke and recreate if needed.

Keep the key in an environment variable (e.g. `FIWANO_API_KEY`); never hardcode or commit it.

**Verify the key works** by listing channels:

```bash
curl https://fiwano.com/api/v1/channels -H "X-API-Key: $FIWANO_API_KEY"
```

A valid key on a fresh account (no channels yet) returns **`200`** with an empty list — this is the success signal that you're authenticated and ready for Step 2:

```json
{ "channels": [], "total": 0 }
```

An invalid or missing key returns `401`. Error shapes and status codes: [`reference/errors.md`](reference/errors.md).

- OpenAPI: `GET /api/v1/channels`

---

## Step 2 — Connect a channel

There are **two ways to connect**, both covered in [`reference/channels.md`](reference/channels.md) — pick the one that matches who owns the account:

- **Your own channel** — connect it in the [portal](https://fiwano.com) (Channels → Connect), no code. Best when you operate the accounts yourself.
- **Your end-users' channels** — an embedded OAuth flow you drive from your app: create a setup URL, the user completes Meta login inside it, and you exchange the returned one-time `code` for a `channel_id`. The `redirect_uri` must be whitelisted first.

OpenAPI operations for this step:

- Manage channels: `GET /api/v1/channels`, `GET /api/v1/channels/{channel_id}`, `PATCH /api/v1/channels/{channel_id}`, `DELETE /api/v1/channels/{channel_id}`
- Embedded connect flow: `POST /api/v1/channels/setup-url`, `POST /api/v1/channels/exchange-code`
- Redirect-URI whitelist: `GET /api/v1/redirects`, `POST /api/v1/redirects`, `DELETE /api/v1/redirects/{redirect_id}`

**Success:** you have a `channel_id` (the embedded flow returns it straight from `exchange-code`), and `GET /api/v1/channels` now lists the channel with `"is_active": true` — it can send and receive. That `channel_id` is what you pass to every send and receive call from here on.

---

## Step 3 — Receive messages

Replying to inbound is Fiwano's core use case, so set up receiving **before** sending. Two parts:

**1. Enable events on the channel.** Delivery is opt-in — by default no events are delivered. Set `webhook_events` (and a `webhook_url`) on the channel, and enable **only the events you actually handle**. The per-channel event list and how to configure it are in [`reference/channels.md`](reference/channels.md#webhook-events).

**2. Handle the webhook.** Fiwano POSTs each enabled event to your `webhook_url`. Read [`reference/webhooks.md`](reference/webhooks.md) for payload shapes and signature verification. Your endpoint **must verify the signature** and **respond HTTP 2xx within ~5 seconds** — otherwise Fiwano retries with backoff and emails you (see [Retry Policy](reference/webhooks.md#retry-policy)).

Webhooks are the **outbound** contract (payloads we POST to you), so they live in the prose, not in the `/api/v1` OpenAPI. From a handler you'll often also call:

- `PATCH /api/v1/channels/{channel_id}` — set or update `webhook_events` / `webhook_url`
- `GET /api/v1/media/{media_id}` — download received media
- `GET /api/v1/channels/{channel_id}/profile/{user_id}` — look up the sender's profile

**Success:** message your connected channel from a real device; your endpoint receives a `message.received` webhook with a valid signature and returns 2xx. You're now receiving.

---

## Step 4 — Send a message

With receiving in place, send outbound. The everyday case is a **free-form reply within the 24-hour window** after a user messages you — plain text or media, no approval needed. Read [`reference/sending-messages.md`](reference/sending-messages.md).

- Send: `POST /api/v1/messages/send` (text), `POST /api/v1/messages/send-media`

**Success:** the call returns an accepted message with an id; if you enabled delivery events in Step 3, you'll then receive `message.sent` / `message.delivered` webhooks tracking it.

That covers the core loop — connect, receive, reply. The sections below are optional.

---

## WhatsApp templates — messaging outside the 24-hour window (optional)

Free-form messages only reach a user **inside** the 24-hour window. To start a conversation, or to reply after the window has closed, WhatsApp requires a pre-approved **template**. Skip this section if you only ever reply within the window.

Read [`reference/templates.md`](reference/templates.md) for the create/review lifecycle, then send the approved template.

- Send a template: `POST /api/v1/messages/send-template`
- Manage templates: `GET /api/v1/channels/{channel_id}/templates`, `POST /api/v1/channels/{channel_id}/templates`, `GET|PUT|DELETE /api/v1/channels/{channel_id}/templates/{template_id}`

---

## Prefer no-code? Use the n8n nodes

If you build on **n8n**, you don't have to call the API by hand — the verified [`n8n-nodes-fiwano`](https://n8n.io/integrations/fiwano/) package wraps this whole API as native nodes, and the same four steps map directly onto them:

- **Authenticate** → add the *Fiwano API* credential (your `mip_live_` key).
- **Connect a channel** → the **Fiwano** node, *Channel* operations (or connect in the portal).
- **Receive** → the **Fiwano Trigger** node gives you a webhook URL; set it on the channel and choose your events. It verifies the signature for you.
- **Send** → the **Fiwano** node, *Message → Send Text / Media / Template*.

Start with [`reference/n8n.md`](reference/n8n.md). Everything in this playbook about the message model, the 24-hour window, limits and billing applies just the same.

---

## More reference

- Limits & capabilities (rate limits, messaging windows, media limits, license tiers): [`reference/capabilities.md`](reference/capabilities.md)
- Billing & subscription state of a channel: [`reference/subscriptions.md`](reference/subscriptions.md)
- Background reading (pricing, the 24-hour window, ways to connect): [`reference/guides.md`](reference/guides.md)
- Exact contract for every endpoint: [`reference/openapi.yaml`](reference/openapi.yaml)

## Good to know

- Prefer [`reference/openapi.yaml`](reference/openapi.yaml) for exact field names, types and status codes; the prose explains the flow.
- Base URL `https://fiwano.com`; authenticate every request with the `X-API-Key` header.
- Never put real API keys, tokens or webhook secrets into code you commit.
