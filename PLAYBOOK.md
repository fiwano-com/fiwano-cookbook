# Fiwano Integration Playbook

Fiwano puts **WhatsApp, Instagram and Messenger** behind one REST API. The core
loop is simple: receive a message via webhook, reply through the API. This
playbook is the entry point to the cookbook — it points you at the step-by-step
quickstart and maps out where everything else lives.

> Base URL is `https://fiwano.com`. Never commit real API keys, tokens or webhook secrets.

## Start here: the quickstart

**[`reference/quickstart.md`](reference/quickstart.md)** is the step-by-step
integration guide — **four clear steps** (authenticate, connect a channel,
receive a message, reply) plus an optional fifth for messaging outside the
24-hour window. Each step says what to do, names the OpenAPI operations it uses,
and links to the deeper reference. Read it top to bottom.

The prose explains *how*; [`reference/openapi.yaml`](reference/openapi.yaml) is
the exact, machine-readable contract (every field, type and status code).

## Reference documentation

What each file covers, so you (or the agent) can jump straight to the right one:

| File | What's inside |
|------|---------------|
| [`reference/quickstart.md`](reference/quickstart.md) | The step-by-step integration plan — **start here** |
| [`reference/channels.md`](reference/channels.md) | Connect, manage and reconnect WhatsApp, Instagram and Messenger channels (portal and embedded OAuth), and the per-channel webhook event list |
| [`reference/webhooks.md`](reference/webhooks.md) | Receiving messages and statuses, signature verification, retry policy, downloading media, sender profiles |
| [`reference/sending-messages.md`](reference/sending-messages.md) | Sending text, media and WhatsApp template messages |
| [`reference/templates.md`](reference/templates.md) | Creating, managing and reviewing WhatsApp message templates |
| [`reference/capabilities.md`](reference/capabilities.md) | Channel capabilities, license tiers, rate limits, messaging windows and media limits |
| [`reference/subscriptions.md`](reference/subscriptions.md) | A channel's subscription state and the billing lifecycle |
| [`reference/errors.md`](reference/errors.md) | Error format and HTTP status codes |
| [`reference/n8n.md`](reference/n8n.md) | The verified n8n community node — same channels, events and operations, no code |
| [`reference/guides.md`](reference/guides.md) | Background reading (pricing, the 24-hour window, ways to connect) — links to the full articles |
| [`reference/openapi.yaml`](reference/openapi.yaml) | The exact machine-readable contract for every endpoint |

## Prefer no-code? Use the n8n nodes

If you build on **n8n**, you don't have to call the API by hand — the verified
[`n8n-nodes-fiwano`](https://n8n.io/integrations/fiwano/) package wraps this whole
API as native nodes (credential, channel operations, a trigger node that verifies
signatures, and send text/media/template). The quickstart's four steps map
directly onto them. Start with [`reference/n8n.md`](reference/n8n.md).

## Good to know

- Prefer [`reference/openapi.yaml`](reference/openapi.yaml) for exact field names, types and status codes; the prose explains the flow.
- Base URL `https://fiwano.com`; authenticate every request with the `X-API-Key` header.
- Never put real API keys, tokens or webhook secrets into code you commit.
</content>
