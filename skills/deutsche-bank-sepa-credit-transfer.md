---
name: deutsche-bank-sepa-credit-transfer
description: Initiate, authorise, track and cancel a SEPA Credit Transfer or SEPA Instant Credit Transfer through the Deutsche Bank dbAPI, including PSD2 strong customer authentication and idempotent retries.
api: Deutsche Bank API Program (dbAPI)
provider: deutsche-bank
generated: '2026-09-06'
method: generated
source: openapi/deutsche-bank-dbapi-sepaCreditTransfer-v3.json, openapi/deutsche-bank-dbapi-payments-sepaInstantCreditTransfer-v3.json, openapi/deutsche-bank-dbapi-transactionAuthorization-v1.json
operations:
  - getReachabilityStatus
  - getPaymentStatus
  - getChallengeMethodsV2
  - createChallengeV2
  - verifyChallengeV2
  - switchMethodV2
  - verifyPushTANChallengeV2
scopes:
  - sepa_credit_transfers
  - instant_sepa_credit_transfers
  - bulk_sepa_credit_transfers
  - bulk_instant_sepa_credit_transfers
---

# Move money over SEPA with the Deutsche Bank dbAPI

**This skill spends real money in production.** Do not run any step past section 2 without an
explicit human authorisation for this specific payment.

Note on operationIds: the payment specs declare summaries but **no `operationId`** on the
credit-transfer operations. Address them by method and path, exactly as written below.

## Surfaces

| Scheme | Spec | Path root |
|---|---|---|
| SEPA Credit Transfer | `openapi/deutsche-bank-dbapi-sepaCreditTransfer-v3.json` | `/paymentInitiation/payments/v3/sepaCreditTransfer` |
| SEPA Instant Credit Transfer | `openapi/deutsche-bank-dbapi-payments-sepaInstantCreditTransfer-v3.json` | `/paymentInitiation/payments/v3` |
| SEPA Direct Debit | `openapi/deutsche-bank-dbapi-sepaDirectDebit-v1.json` | `/paymentInitiation/payments/v1/sepaDirectDebit` |

Each has a single and a `bulk` variant.

## 1. Pre-flight (safe, no side effects)

- `getReachabilityStatus` — `GET /creditorAccounts/{iban}/reachabilityStatus` — verifies the
  creditor IBAN can receive a SEPA Instant transfer. Run this before an instant payment; a
  non-reachable IBAN is a guaranteed failure.
- Verification of Payee is available after initiation via `GET .../{paymentId}/vopDetails`.
  Error `6532` means the customer is not eligible to opt out of VoP.

## 2. Initiate

`POST /` (single) or `POST /bulk`.

Required headers:

- `Correlation-Id` — always.
- `idempotency-id` — **required, uuid**. "Must be present during retries to avoid multiple
  processing of the same request." Generate ONE uuid per logical payment and reuse it for
  every retry of that payment. Never mint a new one to get past an error.

Validation rules the contract enforces (see `errors/deutsche-bank-error-codes.yml`):

- `6533` — charge bearer must be `SLEV`.
- `6531` — amount may carry at most 2 decimal places.
- `6527` / `6528` / `6529` — on bulk, the control sum and the group-header / payment-information
  transaction counts must match the transactions actually supplied.
- `6526` — `createDateTime` must match `yyyy-MM-dd'T'HH:mm:ss`.

## 3. Strong customer authentication

Payment writes carry an `OTP` header. Source it from the Transaction Authorization API
(`openapi/deutsche-bank-dbapi-transactionAuthorization-v1.json`):

1. `getChallengeMethodsV2` — `GET /challenges/methods` — which 2FA methods this customer has.
2. `createChallengeV2` — `POST /challenges` — start the challenge.
3. `switchMethodV2` — `PATCH /challenges/{id}/method` — if the customer wants a different method.
4. `verifyPushTANChallengeV2` — `GET /challenges/{id}` — poll a PushTAN challenge.
5. `verifyChallengeV2` — `PATCH /challenges/{id}` — submit the customer's response.

Error `111` means the 2FA method is not ACTIVE; `17` means the OTP was invalid; `16` is an
invalid challenge response. `6534` means the MFA session expired — re-initiate the payment.

If the second factor fails after initiation, use `PATCH /{paymentId}` ("Second factor retry"),
which also takes `idempotency-id`.

## 4. Track

- `GET /{paymentId}/status` — scheme status.
- `GET /{paymentId}` — full detail.

## 5. Reverse

- `DELETE /{paymentId}` (and `DELETE /bulk/{paymentId}`) — "Cancel a previously initiated SEPA
  Credit Transfer." Takes `idempotency-id`.

**Deutsche Bank does not publish a cancellation window for the payment surfaces.** Do not tell
a user "you have N hours to cancel" — the contract does not say so. Attempt the cancel and read
the response. On an instant transfer, assume the window is effectively zero once settled.

## Idempotency and retries

- `409` with code `6506` — "The IdempotencyId already being used." The original request is
  already in flight. **Poll the status operation. Do not resubmit with a fresh uuid** — that is
  how you send the payment twice.
- `500` — retry with the SAME `idempotency-id`.
- There is no `429` and no `Retry-After` in this estate.
