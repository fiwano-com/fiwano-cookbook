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

### Webhook auto-setup

The trigger can wire its own webhook onto your channels, so you don't have to call **Update Webhook** by hand. Pick a **Webhook Auto-Setup** mode and attach a Fiwano API credential. The auto modes (**All Active Channels** / **Specific Channel**) need it to call the API — if it's missing, activation fails with a clear error. In **Manual** the credential is optional, used only to read a default webhook secret:

| Mode | What happens on activation | On deactivation |
|---|---|---|
| **All Active Channels** | Points every active channel that **isn't already wired elsewhere** (WhatsApp + Instagram + Facebook) at this trigger — one workflow handles all three. Channels already pointing at another URL are **left untouched**. | Clears the webhook on the channels that still point at this trigger. |
| **Specific Channel** | Points one **Channel ID** at this trigger — **takes it over** even if it already has a webhook. | Clears that channel's webhook (only if it still points here). |
| **Manual** *(default)* | Nothing — you set `webhook_url` yourself via **Exchange OAuth Code** / **Update Webhook**. No credential needed. | Nothing. |

The trigger's selected **Event Types** are registered as the channel's `webhook_events` (events that don't apply to a channel type are ignored — e.g. `message.sent`/`message.failed` on Instagram). Channels start with no events enabled, so auto-setup turns them on for you.

**Webhook secret.** Set a **Webhook Secret** to verify incoming signatures (HMAC-SHA256; mismatches are rejected with HTTP 401) and, in auto-setup, to register on your channels. You can set it in two places: the trigger's own **Webhook Secret** field, or — to reuse one secret everywhere — the **Webhook Secret** field on the Fiwano API credential. The trigger's field wins; if it's empty, the credential's secret is used. That same credential secret also backs the **Exchange OAuth Code** and **Update Webhook** operations when you leave their secret empty. Leave both empty to skip verification (not recommended in production).

**When it runs:** only on workflow **activation / deactivation** (and when n8n restarts active workflows) — **never per message**, so it adds no overhead to message handling. A few points to keep in mind:

- **Deactivating removes the webhook** from the channels that point at this trigger. This only clears the webhook URL — it does **not** delete the channel, messages, or any data. While deactivated, Fiwano still stores inbound messages but doesn't relay them; reactivate to resume.
- **All Active Channels skips channels silently.** A channel already pointing at another URL is left alone and the workflow still activates without an error. So if one channel isn't responding, check whether its webhook points somewhere else — clear it or use **Specific Channel** to take it over.
- **Clean up before removing.** Deactivate the workflow (don't just delete it, and don't remove the credential first) so the trigger can clear the webhook. If cleanup can't run, a channel keeps pointing at an inactive n8n URL — Fiwano then logs delivery failures and emails you until you clear it (via **Update Webhook** or the portal).
- Connect a **new channel** after activating? Re-activate the workflow (toggle off/on) so the trigger wires it.
- Two **All Active Channels** workflows won't fight over a channel — whichever claims an unwired channel first owns it; the other leaves it alone. To move a channel deliberately, clear its webhook or use **Specific Channel**.
- Your n8n must be **publicly reachable** — Fiwano delivers webhooks over the internet to the URL the trigger registers.

### Example workflows

Two ready-to-import workflows are in the [GitHub repository](https://github.com/fiwano-com/n8n-nodes-fiwano/tree/main/workflows). Use them in order:

1. **Connect a Channel** — generate a Meta setup link per channel and capture the connected `channel_id` automatically via a webhook callback. (You can also connect channels in the [Fiwano portal](https://fiwano.com).)
2. **Universal Auto-Responder** — one trigger answers **every** message across WhatsApp, Instagram and Facebook: echoes text and replies to attachments with file details. The unified ping-pong pattern — **needs at least one connected channel** (step 1).

Import them from the editor (**Workflows → Import from File…**) or the CLI.
