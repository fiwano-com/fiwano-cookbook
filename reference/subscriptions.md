# Subscriptions & Billing

Every channel returned by `GET /api/v1/channels` carries a `subscription` object
describing its current billing state. This page explains what those states mean
and how they change over a channel's lifecycle. For the field types, see the
**[API Reference](openapi.yaml)**.

A channel can **send and receive messages only while its subscription is
`active`.** When it is not, send/receive calls are rejected until a license is
(re)attached.

### The subscription object

```json
"subscription": {
  "status": "active",
  "source": "paddle",
  "tier": "pro",
  "expires_at": "2025-02-15T10:30:00",
  "auto_renew": true
}
```

- **`status`** — `active`, `expired`, `canceled`, or `none` (no license bound;
  the channel cannot send/receive).
- **`source`** — where the entitlement came from: `trial` (auto-granted on
  signup), `paddle` (paid subscription), or `enterprise` (custom subscription
  provisioned by Fiwano staff, e.g. a partner deal or invoice billing). `null`
  when `status` is `none`.
- **`tier`** — `starter` or `pro`. `pro` is required for media messages and
  WhatsApp template CRUD/send. `null` when `status` is `none`.
- **`expires_at`** — ISO-8601 UTC timestamp when the current period ends. If
  `auto_renew` is `true`, this is the next renewal date; otherwise it is the
  cutoff after which the channel stops working.
- **`auto_renew`** — `true` only for an active Paddle subscription that will renew
  at `expires_at`. Always `false` for trial and Enterprise.

### What the combinations mean

- **Active Paddle subscription** —
  `{status: "active", source: "paddle", auto_renew: true, expires_at: <next renewal>}`.
- **Paddle renewal being retried** —
  `{status: "active", source: "paddle", auto_renew: true, expires_at: <recently in the past>}`.
  While a renewal payment is retried, `status` stays `active` and `expires_at` may
  sit slightly in the past — **service continues during this short grace window.**
  It then resolves to renewed (future `expires_at`) or, if payment keeps failing,
  lapses.
- **Paddle with cancellation scheduled** —
  `{status: "active", source: "paddle", auto_renew: false, expires_at: <cutoff>}`.
  The customer cancelled in Paddle; service continues until `expires_at`, then the
  channel becomes orphaned.
- **Trial** —
  `{status: "active", source: "trial", tier: "pro", auto_renew: false, expires_at: <signup + 7 days>}`.
- **Enterprise** —
  `{status: "active", source: "enterprise", auto_renew: false, expires_at: <agreed term end>}`.
  Renewals are arranged with Fiwano staff before `expires_at`.
- **No active subscription** —
  `{status: "none", source: null, tier: null, expires_at: null, auto_renew: false}`.
  Send/receive will fail; attach a license to restore service.

> **Tip.** Treat `status` as the single source of truth for whether a channel can
> operate. Do not infer it yourself from `expires_at` — during the Paddle grace
> window an `active` channel can legitimately have an `expires_at` in the past.

### Checking available slots

Use `GET /api/v1/subscriptions` when an external service needs to decide whether
it can start a new channel connection flow. The endpoint is read-only and returns
all subscriptions plus aggregate slot availability:

```bash
curl -H "X-API-Key: $FIWANO_API_KEY" \
  https://fiwano.com/api/v1/subscriptions
```

```json
{
  "available_slots": {
    "whatsapp": { "total": 0, "starter": 0, "pro": 0 },
    "instagram": { "total": 1, "starter": 0, "pro": 1 },
    "facebook": { "total": 1, "starter": 0, "pro": 1 }
  },
  "subscriptions": [
    {
      "id": "a1b2c3d4e5f67890",
      "status": "active",
      "source": "trial",
      "tier": "pro",
      "auto_renew": false,
      "assigned_channels": {
        "whatsapp": {
          "channel_id": "1111222233334444",
          "channel_type": "whatsapp",
          "name": "Acme Support",
          "is_active": false
        },
        "instagram": null,
        "facebook": null
      }
    }
  ]
}
```

Use `available_slots.<channel_type>.total > 0` as the signal that a new channel
of that type can be connected. An inactive channel can still occupy a slot
because Fiwano preserves the binding for reconnect. In `available_slots`, `total`
is the sum of the currently free `starter` and `pro` slots for that channel type.
In each subscription, `assigned_channels` shows which channel is assigned to the
subscription for each type; `null` means no channel is assigned there. The reverse
mapping is on the channel itself: `subscription.id` in `GET /api/v1/channels`.

To move a channel to a different subscription, or to release a slot so another
channel of the same type can take it, use `subscription_id` in
`PATCH /api/v1/channels/{channel_id}` — see
[subscription slots](channels.md#subscription-slots).
