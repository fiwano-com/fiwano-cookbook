# WhatsApp Templates

WhatsApp requires **pre-approved templates** to start a conversation outside the
24-hour window (see [Capabilities](capabilities.md#messaging-windows-24h)).
Templates are WhatsApp-only and require a **Pro license**. This page covers
creating, managing and sending them.

### Lifecycle

```
Create → PENDING (Meta review, ~24h) → APPROVED (sendable)
                                      → REJECTED (fix & resubmit)
```

Meta can later pause (`PAUSED`, low quality) or disable (`DISABLED`) an approved
template; a send then fails with `error_code` `132015` / `132016`. Statuses are
refreshed from Meta automatically whenever you read templates. A `REJECTED`
template carries Meta's `rejected_reason`, an approved one its `quality_score`
(`GREEN`, `YELLOW`, `RED`, `UNKNOWN`).

Templates belong to the channel's WhatsApp Business Account (WABA): every number
of the same WABA sees and sends the same templates. Manage them
through these endpoints — full request/response schemas are in the
[API Reference](openapi.yaml):

| Task | Endpoint |
|---|---|
| List (filter by status) | `GET /api/v1/channels/{id}/templates` |
| Get one (components + variable definitions) | `GET /api/v1/channels/{id}/templates/{template_id}` |
| Create (→ submitted to Meta, starts `PENDING`) | `POST /api/v1/channels/{id}/templates` |
| Update components | `PUT /api/v1/channels/{id}/templates/{template_id}` |
| Delete | `DELETE /api/v1/channels/{id}/templates/{template_id}` |

### Creating a template

A template is a `name` + `category` (`MARKETING`, `UTILITY`, or `AUTHENTICATION`)
+ `language` + `components`. `BODY` is required; `HEADER` (text only through the
API; image, video and document headers can be created in the portal), `FOOTER` and
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

<a id="sending"></a>

### Sending a template

`POST /api/v1/messages/send-template` sends an `APPROVED` template. Provide variable
values keyed by component. **Positional** templates (`{{1}}`, `{{2}}`) take arrays:

```bash
curl -X POST https://fiwano.com/api/v1/messages/send-template \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "template_name": "order_confirmation",
    "language": "en_US",
    "recipient": "1234567890",
    "variables": {
      "header": ["Summer Sale"],
      "body": ["Pablo", "ORD-123", "25%"],
      "buttons": [{"index": 0, "value": "promo25"}]
    }
  }'
```

**Named** templates (`{{customer_name}}`) take objects:

```bash
  -d '{
    "channel_id": "a1b2c3d4e5f67890",
    "template_name": "welcome_message",
    "language": "en_US",
    "recipient": "1234567890",
    "variables": {"body": {"customer_name": "Pablo", "order_number": "ORD-123"}}
  }'
```

Omit `variables` entirely if the template has none.

**Media header.** When the template header is an image, video or document
(`variables.header_media` of the template), pass the file as an HTTPS URL in
`header_media_url`. Meta downloads it as in
[media sends](sending-messages.md#response-time), so such a send takes
as long as a media send.

The response is the same as for text and media sends: keep the `message_id` (a
Fiwano UUID) to match delivery webhooks. A template send is never retried
automatically — see [Delivery and retries](sending-messages.md#delivery-and-retries).

<a id="authentication"></a>

### Authentication (one-time codes)

Meta sets the text of an `AUTHENTICATION` template ("*123456* is your
verification code."); you choose the extras and add one copy-code button:

```json
"components": [
  {"type": "BODY", "add_security_recommendation": true},
  {"type": "FOOTER", "code_expiration_minutes": 10},
  {"type": "BUTTONS", "buttons": [{"type": "OTP", "otp_type": "COPY_CODE", "text": "Copy code"}]}
]
```

`FOOTER` is optional. One-tap and zero-tap buttons need an Android app — create
those in WhatsApp Manager; Fiwano sends them like any other template.

Send the code once — it also fills the copy-code button (up to 15 characters):

```bash
  -d '{"channel_id": "a1b2c3d4e5f67890", "template_name": "login_code",
       "language": "en_US", "recipient": "1234567890",
       "variables": {"body": ["482913"]}}'
```

### Buttons

`BUTTONS` can hold `URL`, phone-number and `QUICK_REPLY` buttons. When a
recipient taps a quick-reply button, your webhook receives an ordinary
[`message.received` of `type: "text"`](webhooks.md#button-taps)
whose `text` is the button label, with `reply_to.message_id` equal to the
`message_id` you got from `send-template` — so you know both which button was
chosen and which send it answers. Standalone interactive reply-button and list
messages (outside templates) cannot be sent yet, but taps on them are received
the same way.
