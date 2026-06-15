# n8n Integration

Fiwano is a **verified n8n community node** — listed on [n8n.io/integrations/fiwano/](https://n8n.io/integrations/fiwano/). Use it to build WhatsApp, Instagram and Facebook Messenger automations, AI agent workflows, and chatbots.

### Install

#### From the n8n editor (recommended)

1. Open the nodes panel with **+** or **N**
2. Search for **Fiwano**
3. Select **Fiwano** under **More from the community**
4. Click **Install**

On n8n Cloud, installation may need to be enabled by the instance owner in the Cloud Admin Panel first.

#### Manual fallback (npm)

Use only when in-app installation is unavailable in your environment (e.g. restricted self-hosted setup).

```bash
mkdir -p ~/.n8n/nodes && cd ~/.n8n/nodes
npm install n8n-nodes-fiwano
# Restart n8n
```

For self-hosted Docker: build this package into a custom n8n image — see the [GitHub repository](https://github.com/fiwano-com/n8n-nodes-fiwano) for details.

### Nodes

| Node | Type | Description |
|---|---|---|
| **Fiwano** | Action | Send messages, manage channels, WhatsApp templates, contact profile enrichment, redirect URIs |
| **Fiwano Trigger** | Webhook Trigger | Receive incoming messages and delivery status webhooks with optional HMAC signature verification |

### Action node — operations

| Resource | Operations |
|---|---|
| Message | Send Text, Send Template (WhatsApp), Send Media (image/audio/video/document) |
| Media | Download (fetch a received media file; expires 60 min after the webhook) |
| Channel | Get Many, Get, Generate OAuth URL, Exchange OAuth Code, Update Webhook, Delete |
| Contact | Get Profile (Instagram & Facebook — returns name/username and profile picture; Instagram also follower count) |
| Template | Get Many, Get, Create, Update, Delete (WhatsApp only) |
| Redirect URI | Get Many, Add, Delete |

### Trigger node — events

Starts your workflow for any of these events (filter by event type in node settings):

| Event | Channels |
|---|---|
| `message.received` | WhatsApp, Instagram, Facebook |
| `message.delivered` | WhatsApp, Instagram, Facebook |
| `message.read` | WhatsApp, Instagram, Facebook |
| `message.sent` | WhatsApp |
| `message.failed` | WhatsApp |

HMAC-SHA256 signature verification is built into the trigger node. To enable it, set the channel's `webhook_secret` in the trigger's **Webhook Secret** field — the node then verifies every webhook against that secret and rejects mismatches with HTTP 401. The secret is the one you configured per-channel (via **Exchange OAuth Code** or **Update Webhook**); n8n stores your copy in the node, it is not fetched automatically. Leave the field empty to skip verification (not recommended in production).

### Example workflow

A ready-to-import demo workflow is included in the [GitHub repository](https://github.com/fiwano-com/n8n-nodes-fiwano) covering the core operations: echo bot with profile enrichment, channel management, template CRUD, text and template messaging.
