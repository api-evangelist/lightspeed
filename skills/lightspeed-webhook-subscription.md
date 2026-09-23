---
name: lightspeed-webhook-subscription
description: Subscribe to Lightspeed Retail X-Series change events and keep an external system in sync, including the version-cursor backfill.
api: lightspeed:retail-x-series
generated: '2026-08-27'
method: generated
source: openapi/lightspeed-x-series-openapi.json, asyncapi/lightspeed-webhooks.yml, https://x-series-api.lightspeedhq.com/docs/pagination
operations:
  - post-webhooks
  - get-webhooks
  - get-webhooks-id
  - put-webhooks-id
  - delete-webhooks-webhookId
  - ListSales
  - ListInventoryRecords
scopes:
  - webhooks
---

# Subscribe to Lightspeed Retail (X-Series) events

## What you can subscribe to

Exactly seven event types, and no more:

`sale.update`, `product.update`, `customer.update`, `inventory.update`,
`register_closure.create`, `consignment.send`, `consignment.receive`

Note the shape: these are mostly `*.update`. There is no `sale.create` and no `*.delete`. Treat
`sale.update` as "a sale changed, go read it", not as "a sale was created".

## Steps

1. **Create the subscription** — `post-webhooks`, `POST /webhooks`. Body requires `active`, `type` and
   `url`. Requires the `webhooks` scope.
   The webhook subscription POST is one of the few X-Series endpoints that uses
   `application/x-www-form-urlencoded` rather than JSON — check the request-format guide before you
   send JSON and get a 400.
2. **List what exists** — `get-webhooks`, `GET /webhooks`, before creating: a webhook with the same type
   and URL already registered returns `409 Conflict`.
3. **Pause instead of deleting** — `put-webhooks-id`, `PUT /webhooks/{webhookId}` with `active: false`.
   `delete-webhooks-webhookId` is a hard delete with no restore path.

## Backfill and gap recovery — the important part

Lightspeed publishes no delivery guarantee, no retry policy and no signature scheme for X-Series
webhooks. Do not build a system that assumes every event arrives.

Use the pagination cursor as a change feed instead. Every collection resource carries a `version`
attribute — a globally monotonic integer that increments whenever the resource changes. Store the
highest `version` you have processed, then re-request with `?after=<that version>` to pull everything
modified since. Repeat until `data` comes back empty. Run this on a schedule alongside the webhook
consumer and the webhook becomes a latency optimisation rather than a correctness dependency.

## Verification

There is no HMAC signature on X-Series webhook deliveries. If you need authenticity, use a
hard-to-guess path in the subscription `url` and re-read the resource through the API before acting on
it. Never trust the webhook body alone for anything that moves money.
