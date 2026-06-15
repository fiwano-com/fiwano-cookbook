# WhatsApp Templates

WhatsApp requires **pre-approved templates** to start a conversation outside the
24-hour window (see [Capabilities](capabilities.md#messaging-windows-24h)).
Templates are WhatsApp-only and require a **Pro license**. This page is about
managing them; to *send* an approved template see
[Sending → Template messages](sending-messages.md#template-messages).

### Lifecycle

```
Create → PENDING (Meta review, ~24h) → APPROVED (sendable)
                                      → REJECTED (fix & resubmit)
```

Templates belong to the channel's WhatsApp Business Account (WABA). Manage them
through these endpoints — full request/response schemas are in the
[API Reference](openapi.yaml):

| Task | Endpoint |
|---|---|
| List (filter by status; syncs from Meta by default) | `GET /api/v1/channels/{id}/templates` |
| Get one (components + variable definitions) | `GET /api/v1/channels/{id}/templates/{template_id}` |
| Create (→ submitted to Meta, starts `PENDING`) | `POST /api/v1/channels/{id}/templates` |
| Update components | `PUT /api/v1/channels/{id}/templates/{template_id}` |
| Delete | `DELETE /api/v1/channels/{id}/templates/{template_id}` |

### Creating a template

A template is a `name` + `category` (`MARKETING`, `UTILITY`, or `AUTHENTICATION`)
+ `language` + `components`. `BODY` is required; `HEADER` (text only), `FOOTER` and
`BUTTONS` are optional. Variables are `{{1}}, {{2}}` (positional) or `{{name}}`
(named) — Meta requires `example` values for review.

```bash
curl -X POST https://fiwano.com/api/v1/channels/a1b2c3d4e5f67890/templates \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "order_confirmation",
    "category": "UTILITY",
    "language": "en_US",
    "components": [
      {"type": "BODY", "text": "Hi {{1}}, your order {{2}} is confirmed.",
       "example": {"body_text": [["Pablo", "ORD-123"]]}}
    ],
    "parameter_format": "positional"
  }'
```

### Rules to know

- **Editing an approved template** re-submits it for review (back to `PENDING`)
  and is rate-limited by Meta: **max 10 edits per 30 days, 1 per 24 hours**. You
  can't change the category of an approved template.
- **Deleting an approved template** locks its **name for 30 days** — you can't
  recreate a template with the same name until then (Meta restriction).
- **Creating** is capped at ~100 templates per WABA per hour.

Once a template is `APPROVED`, send it with
[`POST /api/v1/messages/send-template`](sending-messages.md#template-messages).
