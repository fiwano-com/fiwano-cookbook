# Errors

Fiwano reports a problem in one of two ways:

- **The request is refused** with an HTTP `4xx`/`5xx` status — see
  [HTTP errors](#http-errors). A `4xx` from a send endpoint means no message was
  sent.
- **The message fails.** The send endpoints answer `200` with `success: false`
  and `status: "failed"`, or a send that answered `queued` later ends in a
  [`message.failed`](webhooks.md#event-types) webhook. When Meta
  caused it, the reason is in the [send error code](#send-error-codes).

### HTTP errors

The body has a `detail` field, usually a readable string:

```json
{ "detail": "Channel is inactive" }
```

A validation error (`422`) lists the failing fields in `detail`. Rejections you
may want to handle in code carry a structured `detail` with a stable `code`, a
`message`, and usually a `hint`:

| `detail.code` | Status | When |
|---|---|---|
| `text_too_long` | `400` | Text is over the channel's limit; `max_length` and `actual_length` are included |
| `invalid_recipient` | `400` | `recipient` is empty, has no digits, or is not a numeric PSID/IGSID on Messenger/Instagram; `reason` is `empty`, `no_digits` or `not_numeric` |
| `recipient_equals_sender` | `400` | A WhatsApp send is addressed to the channel's own number |
| `invalid_media_request` | `400` | A sticker uses the wrong field for the channel; `reason` is `media_url_required` (WhatsApp), `sticker_id_required` (Messenger) or `sticker_unsupported` (Instagram) |
| `no_free_slot` | `409` | Every subscription slot for this channel type is taken; `occupied_by` lists the channels holding them |

| Status | Meaning | What to do |
|---|---|---|
| `400` | The request cannot be carried out as sent — for example the channel is inactive or must be reconnected | Read `detail`; fix the request or [reconnect the channel](channels.md#reconnecting-an-inactive-channel) |
| `401` | API key missing, malformed or revoked | Send a valid key — see [Authentication](https://fiwano.com/documentation#authentication) |
| `402` | No active subscription, or the feature needs Pro (media, templates) and the channel is on Starter | See [Subscriptions & Billing](subscriptions.md) |
| `403` | The account is inactive, or the OAuth code belongs to another account | Contact support if the account should be active |
| `404` | Not found, or it belongs to another account | Check the ID |
| `409` | No free subscription slot (`no_free_slot`) | Reconnect a channel, release a slot or add a subscription — see [subscription slots](channels.md#subscription-slots) |
| `410` | Inbound media has expired | Download within 60 minutes — see [Downloading inbound media](webhooks.md#downloading-inbound-media) |
| `422` | A field is missing, has the wrong type or breaks a constraint | Read the field list in `detail` |
| `429` | Rate limit — per channel on sends, per API key on everything else | Retry after `Retry-After` — see [rate limits](capabilities.md#rate-limits) |
| `502` | Meta failed on a sender-profile or template-management call (send endpoints never return it) | Retry later |
| `503` | Temporarily unavailable | Retry after `Retry-After` |

Do not rely on the body of a `5xx`; use the status and `Retry-After`. Codes per
endpoint are in the [API Reference](openapi.yaml).

### Send error codes

When Meta rejects a message, its error code reaches you unchanged:

- in the send response, as `error_code` next to `status: "failed"`;
- in a `message.failed` webhook, as `errors[].code`, with `error_subcode` and
  `error_data.details` when Meta gave them.

`error` carries the readable reason: Meta's own text, plus a hint from Fiwano
where Meta's text does not explain the problem.

The *Retried* column applies to text and media sends: such a send answers
`queued` and Fiwano retries it — see
[Delivery and retries](sending-messages.md#delivery-and-retries).
`send-template` never retries: there every error returns `failed`, with Meta's
code in `error_code`.

| Code | Meaning | Retried | What to do |
|---|---|---|---|
| `10` | Meta refuses the action. There are four unrelated causes | no | See [Error 10](#error-10) |
| `100` | Invalid parameter. Meta uses it for unrelated causes, including **a file above the [size cap](capabilities.md#outbound-media-size)** and a PSID/IGSID that is not a user of this Page | no | Read `error`; check `media_url`, `media_type`, file size and `recipient` |
| `190` | Fiwano's access to the account was revoked or has expired. The channel is marked *Action required* ([`health`](channels.md#channel-health) in the API) and the owner is emailed | no | [Reconnect the channel](channels.md#reconnecting-an-inactive-channel) |
| `200` | WhatsApp: Meta does not allow this account to send. Not a token problem: the channel stays connected and keeps receiving | no | Check the WhatsApp Business account in Meta Business Settings |
| `368` | The account is temporarily blocked for policy violations | no | Resolve it in Meta Business Manager |
| `551` | Messenger/Instagram: this person is not receiving your messages — they blocked the Page, closed the chat, restricted business messages, or never messaged the Page | no | Nothing to fix on your side; do not resend automatically |
| `803` | The recipient does not exist or is unavailable | no | Check the recipient ID |
| `131008`, `131009` | A required parameter is missing, or a value is not valid for this channel | no | Fix the request |
| `131026` | WhatsApp: undeliverable — the number is not on WhatsApp, or the recipient has not accepted WhatsApp's terms or uses an old app version | no | Check the number; do not resend automatically |
| `131042` | A payment problem on the WhatsApp Business account in Meta — no payment method, a credit line over its limit, the account suspended. This is Meta's billing, not your Fiwano subscription | **yes** | Fix billing for the WhatsApp Business account — see [Meta's help article](https://www.facebook.com/business/help/2225184664363779) |
| `131047` | WhatsApp: more than 24 hours since the customer last wrote | no | Send an approved [template](templates.md#sending) |
| `131049`, `131050`, `130472` | WhatsApp marketing template not delivered: Meta's per-user marketing limit, the user stopped your marketing messages, or Meta held it back for an experiment | no | Do not resend automatically |
| `131051` | This message type is not supported on the channel | no | See [channel capabilities](capabilities.md#channel-capabilities) |
| `131052` | Meta could not download `media_url` | no | The URL must answer `200` with a `Content-Type` matching `media_type`, and its signature must still be valid |
| `131053` | Meta could not process the media | **yes**, unless the file itself is wrong | Usually temporary. A WebP sent as `image`, or a sticker that breaks the [sticker rules](sending-messages.md#stickers), fails at once with a hint |
| `131056` | WhatsApp: too many messages to the same recipient in a short time | **yes** | Slow down for that recipient |
| `131057` | WhatsApp Business account in maintenance mode, for example a throughput upgrade | **yes** | Nothing; it is temporary |
| `132000`, `132001`, `132012`, `132015`, `132016` | WhatsApp template problem: variables do not match the template (`132000`, `132012`), the template is missing or not approved in this language (`132001`), paused (`132015`) or disabled (`132016`) by Meta | no | Fix the variables, or check the template in WhatsApp Manager — see [WhatsApp Templates](templates.md) |
| `133010` | The WhatsApp number is not registered on the WhatsApp Business Platform: the *WhatsApp Business App* connection was not completed | no | Reconnect choosing *WhatsApp Business App* and finish the step in the app — see [Reconnecting a channel](channels.md#reconnecting-an-inactive-channel) |

Other codes reach you as Meta returns them, and Fiwano retries any code it does
not know to be permanent. Meta's full lists:
[WhatsApp](https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes),
[Messenger and Instagram](https://developers.facebook.com/docs/messenger-platform/error-codes).

#### Error `10`

Meta returns `10` for unrelated causes. The text in `error` tells them apart:

| `error` contains | Cause | What to do |
|---|---|---|
| *outside of allowed window* | Messenger/Instagram: more than 24 hours since the user last wrote | Wait for the user to write again; there is no template workaround — see [messaging windows](capabilities.md#messaging-windows-24h) |
| *another app is controlling this thread* | Messenger/Instagram: Fiwano is not the app in control of the conversation — another connected app, Meta's inbox or Meta AI is | Make Fiwano the *Default routing app* — see [Prerequisites](channels.md#prerequisites) |
| *Meta has temporarily restricted this Page* (Fiwano's hint, sub-code `1893063`) | Messenger/Instagram: Meta restricted the Page from sending for activity against its messaging policies. Some sends may still go through | Check the Page in Account Quality in Meta Business Suite. Reconnecting does not help; Meta lifts the restriction |
| anything else | Meta denies this action for the account | Read `error` and check the account in Meta Business Settings. Not a token problem: the channel stays connected |
