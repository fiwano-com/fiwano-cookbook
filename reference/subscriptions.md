# Subscriptions & Billing

Every channel returned by `GET /api/v1/channels` carries a `subscription` object
describing its current billing state. This page explains what those states mean
and how they change over a channel's lifecycle. For the field types, see the
**[API Reference](openapi.yaml)**.

A channel can **send and receive messages only while its subscription is
`active`.** When it is not, send/receive calls are rejected until a subscription
is (re)attached.

**Naming:** the API calls it a *subscription*; the portal's Billing page calls the
same thing a *license*. One license = one subscription = one slot per channel
type. Trial, Paddle and Enterprise entitlements are all subscriptions here. A
[prepaid subscription](#prepaid-subscriptions) has no slots: it holds any
number of channels and is charged in credits per day: a daily fee plus each channel.

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

- **`status`** — `active`, `expired`, `canceled`, `frozen` (a prepaid
  subscription paused by Fiwano), or `none` (no subscription bound; the channel cannot
  send/receive).
- **`source`** — where the entitlement came from: `trial` (auto-granted on
  signup), `paddle` (paid subscription), `enterprise` (custom subscription
  provisioned by Fiwano staff, e.g. a partner deal or invoice billing), or
  `prepaid` (a [prepaid subscription](#prepaid-subscriptions)). `null` when `status` is
  `none`.
- **`tier`** — `starter` or `pro`. `pro` is required for media messages and
  WhatsApp template CRUD/send. `null` when `status` is `none`.
- **`expires_at`** — ISO-8601 UTC timestamp when the current period ends. If
  `auto_renew` is `true`, this is the next renewal date; otherwise it is the
  cutoff after which the channel stops working. For a prepaid subscription it is
  a forecast of how long the credit balance lasts, not a fixed date.
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
  subscription ends and the channel stops sending and receiving.
- **Trial** —
  `{status: "active", source: "trial", tier: "pro", auto_renew: false, expires_at: <signup + 7 days>}`.
- **Enterprise** —
  `{status: "active", source: "enterprise", auto_renew: false, expires_at: <agreed term end>}`.
  Renewals are arranged with Fiwano staff before `expires_at`.
- **Prepaid** —
  `{status: "active", source: "prepaid", auto_renew: false, expires_at: <balance lasts until>}`.
  It works while its credits cover the next day; see
  [Prepaid subscriptions](#prepaid-subscriptions).
- **No active subscription** —
  `{status: "none", source: null, tier: null, expires_at: null, auto_renew: false}`.
  Send/receive will fail; attach a subscription to restore service.

> **Tip.** Treat `status` as the single source of truth for whether a channel can
> operate. Do not infer it yourself from `expires_at` — during the Paddle grace
> window an `active` channel can legitimately have an `expires_at` in the past.

**When a subscription ends.** Its channels move to another active subscription
of yours with room for them. A channel with nowhere to go stays bound to the
ended subscription, connected but not working, for 3 days and is then
deactivated. If a subscription with room appears in those 3 days — a new one,
an extension, or credits on a prepaid subscription — the channel moves to it
automatically within a minute, without reconnecting, and we email you that it
works again. A Paddle subscription whose renewal stays unpaid is the exception:
its channels that cannot move are deactivated when the payment retries end.

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
      "channels": [
        {
          "channel_id": "1111222233334444",
          "channel_type": "whatsapp",
          "name": "Acme Support",
          "is_active": false
        }
      ]
    }
  ]
}
```

`available_slots.<channel_type>.total` is how many more channels of that type
you can connect right now — use `total > 0` as the signal to start a connection
flow. Each regular subscription adds its free slot of that type (a deactivated
channel still occupies its slot, see
[subscription slots](channels.md#subscription-slots)). A
[prepaid subscription](#prepaid-subscriptions) adds the channels its credits
cover today, within its limit for the type if one is set; its credits are shared
by all types, so the numbers of different types do not add up. `starter` and
`pro` split `total` by tier.
In each subscription, `channels` lists every channel bound to it, connected or
not: at most one per channel type for a regular subscription, any number for a
[prepaid subscription](#prepaid-subscriptions). (`assigned_channels`, the older per-slot
view of the same data, is deprecated — read `channels`. It stays in API v1 for
compatibility and is `null` for a prepaid subscription, which has no slots.) The reverse
mapping is on the channel itself: `subscription.id` in `GET /api/v1/channels`.

By default the list includes expired and canceled subscriptions. To get only
some statuses, pass `status` (repeat it to combine):

```bash
curl -H "X-API-Key: $FIWANO_API_KEY" \
  "https://fiwano.com/api/v1/subscriptions?status=active"
```

The filter narrows `subscriptions` and `total` only; `active_total` and
`available_slots` always describe the whole account.

To move a channel to a different subscription, or to release a slot so another
channel of the same type can take it, use `subscription_id` in
`PATCH /api/v1/channels/{channel_id}` — see
[subscription slots](channels.md#subscription-slots).

### Prepaid subscriptions

A prepaid subscription is for products that connect many channels. Instead of a
billing period it has a balance of **credits**, and it has no slots: any number
of WhatsApp, Instagram and Facebook channels can sit on one subscription.
Prepaid subscriptions are opened by Fiwano on request: open a ticket on the
portal's [Support](https://fiwano.com/support) page and briefly describe your product — what it
does and how many channels you plan to connect, or write to contact@fiwano.com.
If you already pay for monthly licenses, we can move your channels to the
prepaid subscription and credit the unused part of the subscriptions you cancel
as credits.

**Credits.** Credits are units of the Fiwano service, bought in advance in the
portal: **$1 = 100 credits**. They are used only to pay for channels on Fiwano,
do not expire, cannot be transferred to another account and cannot be exchanged
back for money. Every amount in the API is a whole number of credits.

**How it is charged**

- Each UTC day costs a fixed daily fee plus each connected channel, the same
  number of credits for every channel type. A deactivated channel is not
  charged. The fee is shared by all channels, so the price per channel drops as
  you connect more of them.
- The daily fee is charged also when no channels are connected.
  Daily charges start with the first purchase of credits. The prices of your
  subscription are on its page in the portal. To estimate a budget before you
  switch, use the [cost calculator](https://fiwano.com/billing/calculator) in the portal: it also
  shows when monthly subscription licenses cost less for your channels.
- The day is charged in advance at 00:00 UTC. A channel connected or moved to
  the subscription during the day is charged for that day at once. The full day
  is charged however short the channel was connected, and each different
  channel that was on the subscription that day counts.

**Reading a prepaid subscription.** In `GET /api/v1/subscriptions` it has
`source: "prepaid"`, its channels in `channels`, and a `prepaid` object:

```json
{
  "id": "b7c8d9e0f1a23456",
  "status": "active",
  "source": "prepaid",
  "tier": "starter",
  "auto_renew": false,
  "expires_at": "2027-04-13T00:00:00",
  "channels": [
    {
      "channel_id": "1111222233334444",
      "channel_type": "whatsapp",
      "name": "Acme Support",
      "is_active": true
    },
    {
      "channel_id": "5555666677778888",
      "channel_type": "whatsapp",
      "name": "Acme Sales",
      "is_active": true
    }
  ],
  "assigned_channels": null,
  "prepaid": {
    "balance_credits": 6600,
    "daily_cost_credits": 50
  }
}
```

- **`balance_credits`** — the credit balance.
- **`daily_cost_credits`** — credits one day costs with the channels connected now.
- **`expires_at`** — a forecast: until when the balance covers the subscription
  at that cost. It moves as channels are connected or disconnected; `null` while
  no credits were added yet. After the subscription stops it is the moment it
  stopped.
- **`assigned_channels`** is `null`: a prepaid subscription has no slots.
- **`available_slots`** (top level) counts the channels the credits cover today
  — see [Checking available slots](#checking-available-slots).

The full history of purchases and charges is on the subscription's page in the
portal (Billing).

**When the credits run out.** If the balance does not cover the next day, that
day is not charged and the subscription becomes `expired`; the remaining credits
stay. Its channels stop sending and receiving: a send answers `402`
`insufficient_credits`. Channels move to a regular
subscription with a free slot when you have one; the rest stay connected for 3
days and are then deactivated — unless a subscription with room appears in the
meantime: they move to it automatically. We email you a week, 3 days and 24 hours
before the balance runs out. Buying credits that cover the day makes the
subscription `active` again, and channels that are still connected resume
without reconnecting.

**Frozen subscription.** On request, Fiwano staff can freeze a prepaid
subscription with no channels on it: its status becomes `frozen`, daily
charges stop and the credits stay. Channels cannot be connected to it while it
is frozen. When it is unfrozen it becomes `active` and the current day is
charged at once.

**Connecting and moving channels.** A new channel uses a free slot of a regular
subscription first and a prepaid subscription otherwise. To put a channel on a
prepaid subscription explicitly, send its ID as `subscription_id` in
`PATCH /api/v1/channels/{channel_id}`.

When a prepaid subscription cannot take a channel — on `PATCH` or on
`POST /api/v1/channels/setup-url` — the error `detail` is an object with
`code`, `message` and `hint`:

- `402` `insufficient_credits` — the credits do not cover the channel today, or
  the subscription is stopped. Buy credits in the portal, then retry.
- `409` `channel_limit_reached` — the limit for that channel type is reached.
  Contact support to change it.

```json
{
  "detail": {
    "code": "insufficient_credits",
    "message": "The prepaid subscription does not have enough credits to cover a WhatsApp channel today.",
    "hint": "Buy credits on the subscription's page in the portal (Billing), then retry."
  }
}
```
