---
name: deutsche-bank-account-and-transaction-read
description: Read a Deutsche Bank customer's cash accounts, credit cards and transaction history through the dbAPI, using OAuth 2.0 authorization-code delegated access.
api: Deutsche Bank API Program (dbAPI)
provider: deutsche-bank
generated: '2026-09-06'
method: generated
source: openapi/deutsche-bank-dbapi-cashAccounts-v2.json, openapi/deutsche-bank-dbapi-transactions-v2.json, openapi/deutsche-bank-dbapi-creditCards-v1.json, openapi/deutsche-bank-dbapi-creditCardTransactions-v1.json
operations:
  - getCashAccounts
  - getBrand
  - getCashAccountTransactions
  - getCashAccountTransactionById
  - getCreditsCards
  - getCreditCardTransactions
scopes:
  - read_accounts_list
  - read_accounts
  - read_transactions
  - read_credit_cards_list_with_details
  - read_credit_card_transactions
  - offline_access
---

# Read Deutsche Bank accounts and transactions

Read-only flow. Nothing here moves money.

## Before you start

- Sandbox base: `https://simulator-api.db.com/gw/dbapi`
- Production base: `https://api.db.com/gw/dbapi` (norisbank: `https://api.norisbank.de/gw/dbapi`, Postbank: `https://api.postbank.de/gw/dbapi`)
- Every request MUST carry a `Correlation-Id` header. Generate one per call and log it — on an error the response body also returns `messageId`, and support needs both.

## 1. Get a delegated access token

Use the `api_auth_code` scheme (OAuth 2.0 authorization code, PKCE S256 supported).

- authorize: `https://simulator-api.db.com/gw/oidc/oauth2/authorize`
- token: `https://simulator-api.db.com/gw/oidc/oauth2/token`

Request only the scopes the steps below need. Add `offline_access` if you need a refresh token.

## 2. List the customer's cash accounts

`getCashAccounts` — `GET /banking/cashAccounts/v2/` — scope `read_accounts_list`.

Returns the accounts with IBAN and balance. Take the `iban` of the account you want.

Optional: `getBrand` — `GET /banking/cashAccounts/v2/brand?iban={iban}` — scope `read_brand` — resolves which brand/tenant an IBAN belongs to. This one is client-credentials only.

## 3. Read transactions

`getCashAccountTransactions` — `GET /banking/transactions/v2/` — scope `read_transactions`.

- The API serves up to **13 months** of history by default.
- `bookingDateFrom` / `bookingDateTo` bound the window. If the call was NOT made with a PSD2-compliant strong customer authentication, the maximum look-back is shortened by the PSD2 day count and an over-long `bookingDateFrom` returns an error — do not hard-code 1980-01-01 outside an SCA'd session.
- Page with `offset` and `limit`; the envelope returns `totalItems`.
- Since release 2026.06 each transaction can carry an optional `uniqueTag` — a stable identifier. Use it for reconciliation and duplicate detection rather than hashing the payload yourself. It is available for the Deutsche Bank and norisbank tenants.

`getCashAccountTransactionById` — `GET /banking/transactions/v2/{transactionId}` — single read.

## 4. Cards

`getCreditsCards` — `GET /cards/creditCards/v1/` — scope `read_credit_cards_list_with_details`.
`getCreditCardTransactions` — `GET /cards/creditCardTransactions/v1/` — scope `read_credit_card_transactions`.

## Error handling

Errors are NOT RFC 9457. The body is `{ "code": <int>, "message": "<string>", "messageId": "<string>" }`.

- `401` — token missing, expired or wrong scope. Re-mint; use the refresh_token grant if you hold one.
- `403` — token is valid but lacks the scope or tenant entitlement.
- `400` — deterministic validation or business-rule failure. Look the numeric `code` up in `errors/deutsche-bank-error-codes.yml`; retrying without changing the request will not help.
- `500` — retry is safe on these read operations.

There is **no** `429`, no `Retry-After` and no rate-limit header anywhere in this estate. Do not build a backoff loop that waits on a header that will never arrive — use a fixed conservative interval.
