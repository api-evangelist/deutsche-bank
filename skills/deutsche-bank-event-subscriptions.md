---
name: deutsche-bank-event-subscriptions
description: Subscribe to Deutsche Bank transaction and investments-order events, receive them at your own notificationURL, and manage the subscription lifecycle through the dbAPI Notification Service.
api: Deutsche Bank API Program (dbAPI)
provider: deutsche-bank
generated: '2026-09-06'
method: generated
source: openapi/deutsche-bank-dbapi-subscriptions-v1.json
operations:
  - transactionsPost
  - transactionsGet
  - transactionsSubscriptionIdPatch
  - transactionsSubscriptionIdDelete
  - subscriptionActivation
  - investmentsOrdersPost
  - investmentsOrdersGet
  - investmentsOrdersSubscriptionIdPatch
  - investmentsOrdersSubscriptionIdDelete
  - investmentSubscriptionActivation
scopes:
  - transaction_notifications
  - investments_orders_status_notification
---

# Subscribe to Deutsche Bank events

Stop polling `getCashAccountTransactions`. The dbAPI Notification Service pushes to a URL you own.

Base: `/notifications/subscriptions/v1` on the tenant gateway.

## 1. Create a subscription

`transactionsPost` — `POST /transactions` — scope `transaction_notifications`.
`investmentsOrdersPost` — `POST /investments/orders` — scope `investments_orders_status_notification`.

Body is `{ filterCriteria, subscriptionDetails }`. `subscriptionDetails` requires:

- `notificationURL` (url) — where Deutsche Bank will POST the notification.
- `subscriptionType` — `one-time` or `recurring`.
- `expirationDate` (date, optional) — when the subscription lapses.

Headers: `Correlation-Id` and `Idempotency-ID` (note the capitalisation on this spec — it is
`Idempotency-ID` here, not `idempotency-id`). These two POSTs are the only replay-protected
operations on this surface.

## 2. Activate

`subscriptionActivation` — `PATCH /{subscriptionId}` — and
`investmentSubscriptionActivation` — `PATCH /investments/{subscriptionId}` — flip `isActive`.

A created subscription is not necessarily live until activated. Read it back with
`transactionsGet` / `investmentsOrdersGet` and confirm `isActive` before relying on delivery.

## 3. Receive

**Deutsche Bank does not publish the notification payload schema.** The subscription contract is
published; the callback body is not. Build the receiver defensively:

- Accept any JSON body and log it verbatim before parsing.
- Treat the notification as a signal, not as the data — re-read the authoritative record with
  `getCashAccountTransactions` or `orderDetails`. Since 2026.06 the transaction `uniqueTag`
  gives you a stable key to dedupe on.
- No retry, backoff or replay policy is published for delivery. Assume at-least-once and make
  your handler idempotent on your side.

Contrast with the Merchant Solutions callback (`openapi/deutsche-bank-merchant-solution-callback-v2_1.json`),
where Deutsche Bank *does* publish the receiver contract, including the `Digest` (SHA-256 of the
body) and `Signature` (HMAC-256 over Digest, `X-RequestDate` and `X-RandomValue`) headers. The
Notification Service has no documented equivalent — verify the caller by other means.

## 4. Amend and delete

- `transactionsSubscriptionIdPatch` — `PATCH /transactions/{subscriptionId}`.
- `transactionsSubscriptionIdDelete` — `DELETE /transactions/{subscriptionId}`.
- Same pair under `/investments/orders/{subscriptionId}`.

Neither PATCH nor DELETE accepts an idempotency key. Read the subscription list back after a
failed delete rather than repeating it blindly.
