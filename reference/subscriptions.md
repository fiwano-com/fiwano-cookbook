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
