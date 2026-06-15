# Fiwano Cookbook

A practical, hands-on home for integrating **Fiwano** — a step-by-step playbook plus the full API reference, to take you from zero to a working **WhatsApp, Instagram & Facebook Messenger** integration.

## What is Fiwano?

Fiwano puts WhatsApp, Instagram and Facebook Messenger behind **one REST API**, so you can add two-way messaging to an external product — a **chatbot, an AI assistant**, a CRM or a helpdesk. The core loop is simple: receive a message via webhook, reply through the API.

- **One API, three channels** — the same request format for WhatsApp, Instagram DM and Messenger; no per-channel logic to build and maintain.
- **Built for AI agents & chatbots** — receive messages on a webhook, process them with your model or agent, send replies through the API. It's a primary use case.
- **No Meta app setup** — connect accounts through our verified Meta Tech Provider app: no developer portal, no app creation, no App Review on your side.
- **Official Meta APIs — no ban risk** — built on the WhatsApp Cloud API, Instagram Messaging API and Messenger API, not browser automation or unofficial clients.
- **Embed in your SaaS** — an OAuth flow lets *your* customers connect *their own* WhatsApp/Instagram/Facebook accounts inside your product; your code never touches their credentials.
- **Reply-friendly economics** — replying to inbound is free across all three channels (within WhatsApp's 24-hour window); a flat subscription with no per-message markup.

We're a verified Meta Tech Provider.

- 📖 Documentation: https://fiwano.com/documentation
- 🔌 Interactive API reference: https://fiwano.com/documentation/api
- 🚀 Create an account & API key: https://fiwano.com — 7-day free trial, no credit card

## Start here: the Playbook

**[`PLAYBOOK.md`](PLAYBOOK.md)** is a step-by-step integration guide in four clear steps — authenticate, connect a channel, receive messages, and reply. Read it top to bottom, or hand the whole cookbook to an AI coding agent (Cursor, Claude Code, etc.) and point it at `PLAYBOOK.md` — the playbook covers both.

## What's inside

| Path | What |
|------|------|
| [`PLAYBOOK.md`](PLAYBOOK.md) | Step-by-step integration guide — **start here** |
| [`reference/`](reference/) | The full API reference: `openapi.yaml` (the machine-readable contract) plus topic docs, mirrored from [fiwano.com/documentation](https://fiwano.com/documentation) |

## Prefer no-code?

There's a verified n8n community node — **[`n8n-nodes-fiwano`](https://n8n.io/integrations/fiwano/)** — that wraps the whole API as drag-and-drop nodes: send messages, receive events with a trigger node, manage channels and templates. The playbook's *n8n* section shows how the same four steps map onto it.

## Coming soon

Runnable code examples (a webhook receiver with signature verification, sending, embedded signup) and a Postman collection. This cookbook grows incrementally — follow the repo to see new additions.

## Questions & feedback

- Questions and how-to → [Discussions](https://github.com/fiwano-com/fiwano-cookbook/discussions)
- A bug or a request → [Issues](https://github.com/fiwano-com/fiwano-cookbook/issues)
- Account, billing or anything private → contact@fiwano.com

## License

[MIT](LICENSE) — use anything here freely in your own projects.
