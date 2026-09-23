---
name: lightspeed-restaurant-online-order
description: Place and pay for an online order against Lightspeed Restaurant K-Series, including the readiness check and idempotency-keyed payment.
api: lightspeed:restaurant-k-series
generated: '2026-08-27'
method: generated
source: openapi/lightspeed-k-series-openapi.json, conventions/lightspeed-conventions.yml, sandbox/lightspeed-sandbox.yml
operations:
  - apeGetOnlineOrderReadiness
  - apeLoadMenus
  - apeGetMenuByIdV2
  - apePlaceToGoOrder
  - apeLocalOrder
  - apeMakePayment
  - apeGetCheck
scopes:
  - orders-api
---

# Place an online order in Lightspeed Restaurant (K-Series)

## Environments

Both hosts are declared in the contract itself:

- Demo: `https://api.trial.lsk.lightspeed.app`
- Production: `https://api.lsk.lightspeed.app`

Build against the demo host first. It is the only published Lightspeed test environment of the four
flagship APIs.

## Steps

1. **Check readiness** — `apeGetOnlineOrderReadiness`, `GET /o/op/1/onlineOrderReadiness`. A location
   can be closed, paused or not configured for online ordering. Check before you show a menu, not after
   the customer has built a basket.
2. **Load the menu** — `apeLoadMenus` (`GET /o/op/1/menu/list`), then `apeGetMenuByIdV2`
   (`GET /o/op/2/menu/load/{menuId}`).
   Use the **V2** operation. `apeGetMenuById` (`/o/op/1/menu/load/{menuId}`) is flagged `deprecated` in
   the published contract.
3. **Place the order** — `apePlaceToGoOrder` (`POST /o/op/1/order/toGo`) for takeaway, or
   `apeLocalOrder` (`POST /o/op/1/order/local`) for in-venue.
   `apeLocalOrder` changed on 2026-08-20: `collectionCode` and `orderCollectionTimeAsIso8601` were
   modified. Re-read the contract before assuming an older integration still matches.
4. **Take payment** — `apeMakePayment`, `POST /o/op/1/pay`. Payment requests to the
   payments-processing service require an **`Idempotency-Key` header** — a unique key per payment
   request. Generate it once per payment attempt and reuse it on every retry of that same attempt.
5. **Read back the check** — `apeGetCheck` (`GET /o/op/1/order/table/getCheck`) or
   `apeCheckLookup` (`GET /o/op/1/order/table/{tableNumber}/getCheck`).

## Reversibility

There is no cancel, void or refund operation for an order in the K-Series contract. Once
`apeMakePayment` succeeds there is no API path back. Say so plainly to any user before executing it —
this is a one-way write.

## Errors

- `403` — commonly a scope problem, and one scope
  (`user-token-by-authorization-code`) is used by the Reservations operations but never declared in the
  OAuth scope map, so it cannot be requested from the reference alone.
- `409` — the resource already exists or the state transition is not allowed. Read current state first.
- `503` — declared on K-Series operations. Back off with jitter.

## Time and money formats

Timestamps are ISO 8601 (the contract names it in 49 field descriptions, including the field name
`orderCollectionTimeAsIso8601`). Currency codes are ISO 4217.
