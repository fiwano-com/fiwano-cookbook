# Compatibility

Fiwano has no version numbers. The API contract is `v1` (`https://fiwano.com/api/v1`), and it has been stable since the public launch in March 2026. The service ships continuously; new capabilities are listed in the [changelog](https://fiwano.com/changelog) (with an Atom feed).

**Every change to `v1` is additive:**

- new endpoints;
- new **optional** request parameters and fields;
- new fields in responses and webhook payloads;
- new webhook event types and new values in open sets such as delivery statuses or error hints.

**What stays fixed:** existing endpoints, field names, types and meanings; the `X-API-Key` authentication; the webhook signature scheme. New webhook event types are **never enabled on your channels without your action** — you opt in per channel via `webhook_events`.

**What your integration needs to do** to stay compatible: ignore fields it does not know and ignore event types it did not enable. Do not treat an unknown field or a new value in an open set as an error.

If a breaking change ever becomes unavoidable, it ships as a new API version alongside `v1`, announced in the changelog and by email in advance. `v1` keeps working.
