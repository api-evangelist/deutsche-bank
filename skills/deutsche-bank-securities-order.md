---
name: deutsche-bank-securities-order
description: Preview, place, amend and cancel a securities order on a Deutsche Bank portfolio through the dbAPI investments surface, using the mandatory preview-signature binding.
api: Deutsche Bank API Program (dbAPI)
provider: deutsche-bank
generated: '2026-09-06'
method: generated
source: openapi/deutsche-bank-dbapi-investments-orders-v1.json, openapi/deutsche-bank-dbapi-investments-securityAccounts-v1.json, openapi/deutsche-bank-dbapi-investments-assets-v1.json
operations:
  - getSecurityAccounts
  - getAssetsV1
  - securityMarketPlaces
  - orderQuotes
  - costInformation
  - estimatedExpenseReport
  - orderEntryPreview
  - orderEntry
  - bulkOrderEntry
  - orderBookList
  - orderDetails
  - changeSecurityOrderPreview
  - orderChange
  - orderDelete
scopes:
  - read_security_accounts_list
  - read_assets
  - order_securities
---

# Place a securities order at Deutsche Bank

**This skill trades real securities in production.** Every write below needs an explicit human
authorisation and a second factor.

This is the one surface in the Deutsche Bank estate that gives an agent a genuine rehearsal
step, and it is not optional: the preview issues a `previewSignature` that the real order entry
then **requires** as a header. You cannot place an order you have not previewed.

## 1. Locate the portfolio

- `getSecurityAccounts` — `GET /investments/securityAccounts/v1/` — scope
  `read_security_accounts_list`. Take `securityAccountId`.
- `getAssetsV1` — `GET /investments/assets/v1/` — scope `read_assets` — current holdings.

## 2. Rehearse (no side effects)

- `securityMarketPlaces` — `GET /investments/orders/v1/securityMarketPlaces` — which venues are
  available for the instrument.
- `orderQuotes` — `GET /investments/orders/v1/quotes`.
- `costInformation` — `POST /investments/orders/v1/estimatedExpense` — MiFID cost disclosure.
- `estimatedExpenseReport` — `POST /investments/orders/v1/estimatedExpenseReport`.
- `orderEntryPreview` — `POST /investments/orders/v1/preview` — **returns the
  `previewSignature`**. Show the preview to the human. This is the decision point.

## 3. Place the order

`orderEntry` — `POST /investments/orders/v1/` — scope `order_securities`.

Headers: `Correlation-Id`, `previewSignature` (from step 2), `OTP` (from the Transaction
Authorization API — see the SEPA skill for the challenge flow), and `idempotency-id` (uuid).

`bulkOrderEntry` — `POST /investments/orders/v1/bulk` — same headers.

Error `102` means `externalOrderReference` or `securityAccountId` is invalid; `111` means the
2FA method is not ACTIVE.

## 4. Track

- `orderBookList` — `GET /investments/orders/v1/`.
- `orderDetails` — `GET /investments/orders/v1/{orderId}`.

## 5. Amend

- `changeSecurityOrderPreview` — `PATCH /investments/orders/v1/{orderId}/preview` — rehearse the change.
- `orderChange` — `PATCH /investments/orders/v1/{orderId}` — requires `OTP`.

Note: `orderChange` and `orderDelete` do **not** accept an `idempotency-id`. A retry of either
is not replay-protected — read the order status before retrying rather than firing again.

## 6. Cancel — the one reversal with a published window

`orderDelete` — `POST /investments/orders/v1/{orderId}/cancel` — requires `OTP`.

Deutsche Bank states the window explicitly: *"This API tries to cancel an existing order. For
that, the order must be in status ACCEPTED but not executed yet. On successful cancellation,
the status is updated to CANCELLED."*

So: check `orderDetails` for status `ACCEPTED` before promising a user the order can be pulled.
Once executed, there is no reversal operation on this surface.
