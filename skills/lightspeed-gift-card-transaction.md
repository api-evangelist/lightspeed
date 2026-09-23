---
name: lightspeed-gift-card-transaction
description: Issue, redeem, reload and reverse Lightspeed Retail X-Series gift cards safely, using the client_id idempotency key.
api: lightspeed:retail-x-series
generated: '2026-08-27'
method: generated
source: openapi/lightspeed-x-series-openapi.json, https://x-series-api.lightspeedhq.com/docs/gift_cards, conventions/lightspeed-conventions.yml
operations:
  - CreateGiftCard
  - CreateGiftCardTransaction
  - FindGiftCard
  - FindGiftCardByNumber
  - FindGiftCardByTransactionId
  - ReverseGiftCardTransaction
  - VoidGiftCard
  - ListGiftCards
scopes:
  - gift_cards:read
  - gift_cards:write:issue
  - gift_cards:write:redeem
---

# Gift cards in Lightspeed Retail (X-Series)

This is the one Lightspeed surface with a documented, mandatory idempotency mechanism. Use it.

## The idempotency rule

Every **reload** and **redeem** transaction **must** carry a `client_id` in the request body — an
external reference number, transaction id or UUID that is unique per transaction. Lightspeed checks it:
if a second transaction arrives with the same `client_id` and the same transaction details, it is not
applied, and the original transaction is returned in its place. That is what makes a retry safe.

`client_id` is a **body field, not a header**. There is no `Idempotency-Key` header on X-Series.
Lightspeed does not publish how long a `client_id` is remembered.

## Steps

1. **Issue** — `CreateGiftCard`, `POST /gift_cards`. Body may include `amount`, `number` and optionally
   `expires_at` in ISO-8601 (`YYYY-MM-DDTHH:MM:SS+00:00`). Expiry must first be enabled in the
   Lightspeed Retail UI; if you omit `expires_at` and expiry is configured, Lightspeed calculates it.
   Requires `gift_cards:write:issue`.
2. **Redeem** — `CreateGiftCardTransaction`, `POST /gift_cards/{card_number}/transactions` with
   `type: "REDEEMING"`, a **negative** `amount`, and a `client_id`. Requires `gift_cards:write:redeem`.
3. **Reload** — same endpoint, positive `amount`, and again a `client_id`.
4. **Check balance** — `FindGiftCard` (`GET /gift_cards/{number}`), `FindGiftCardByNumber` or
   `FindGiftCardById`. Requires `gift_cards:read`.

## Reversal

- `ReverseGiftCardTransaction` — `DELETE /gift_cards/transactions/{transaction_id}`. On success a new
  transaction with status `REVERSING` is appended. **Only transactions of type `REDEEMING` can be
  reversed** — a reload cannot be undone this way.
- `VoidGiftCard` / `VoidGiftCardById` / `VoidGiftCardByNumber` set the balance to zero and the status to
  `VOIDED`. This is a card-level action, not a transaction-level one.

No time window is published for either. Do not state one.

## Errors

- `422` — commonly an insufficient balance on the card. Do not retry unchanged.
- `403` — the redeem and issue scopes are separate; holding one does not grant the other.
- `429` — see the rate-limit rules in `conventions/lightspeed-conventions.yml`.
