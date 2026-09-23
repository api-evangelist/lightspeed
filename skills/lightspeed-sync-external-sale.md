---
name: lightspeed-sync-external-sale
description: Push a completed sale from an external channel into Lightspeed Retail X-Series and confirm it landed, without creating duplicates.
api: lightspeed:retail-x-series
generated: '2026-08-27'
method: generated
source: openapi/lightspeed-x-series-openapi.json, conventions/lightspeed-conventions.yml, scopes/lightspeed-scopes.yml
operations:
  - CreateSale
  - GetSaleByID
  - ListSales
  - initReturnSale
scopes:
  - sales:write
  - sales:read
  - users:read
---

# Sync an external sale into Lightspeed Retail (X-Series)

Use this when an order completed somewhere else — a marketplace, a kiosk, your own storefront — and the
retailer's Lightspeed account has to reflect it.

## Before you start

- Resolve `domain_prefix` first. The base URL is templated:
  `https://{domain_prefix}.retail.lightspeed.app/api/2026-07`. The prefix arrives in the OAuth callback
  alongside the authorization code, and it is also returned in every token response.
- Your access token must carry `sales:write`. Token responses echo the scopes actually granted; if
  `sales:write` is not in that list, stop and re-authorize rather than calling and reading a 403.
- Set both `Content-Type` and `Accept` to `application/json`.

## Steps

1. **Send an idempotency identifier.** Populate `request_id` on the `CreateSale` body with a stable
   identifier from your own system (the external order id works well). Lightspeed echoes it back in the
   response. This is the only duplicate protection the sales surface offers — there is no
   `Idempotency-Key` header on X-Series.
2. **`CreateSale`** — `POST /sales`. Build the sale with its line items and payments.
3. **Confirm with `GetSaleByID`** — `GET /sales/{sale_id}` using the id from the create response. Do
   not treat a network timeout as a failure: re-send the same `CreateSale` with the same `request_id`,
   or search first with `ListSales`, before assuming nothing was written.
4. **Reconcile with `ListSales`** — `GET /sales`. Page with the `after` cursor: send no `after` on the
   first call, then re-request with the `version.max` value from the previous page until `data` comes
   back empty.

## If you have to undo it

`initReturnSale` — `POST /sales/{sale_id}/actions/return` — initializes a return against an existing
**closed** sale and gives you back a new SAVED return sale; you then add refund payments to finalize it.
It needs `sales:write` and `users:read`.

**Lightspeed does not publish a time window for returns.** Do not tell a user that a sale can be
reversed "within N days" — that number does not exist in Lightspeed's documentation. Confirm with the
retailer before promising reversibility.

## Errors you will actually hit

- `429` — read `Retry-After`, which on X-Series is an **RFC1123 HTTP-date, not a number of seconds**.
  Queue the call; do not sleep between every request. The limit is `300 x registers + 50` per 5-minute
  window.
- `403` — the token is valid but the scope is missing. Compare against `scopes/lightspeed-scopes.yml`.
- `422` — semantically rejected (insufficient balance, invalid combination). Do not retry unchanged.
- `400` since the 2026-07 release — postal code validation was tightened across 40 countries, so
  addresses that used to save may now be rejected.

## Silent behaviour to warn the user about

Lightspeed automatically redacts anything it detects as credit-card data on customer names, company
name, note, and address fields — and returns `200 OK` while doing it. A value you write can differ from
the value you read back, with no error. Never round-trip user text through those fields assuming
fidelity.
