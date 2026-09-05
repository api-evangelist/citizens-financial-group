---
name: citizens-transfer-between-own-accounts
description: Move funds same-day between two accounts the same client holds at Citizens, using the
  Account Transfer API - a single internal transfer with no batching, no future dating and no cancel.
api: Citizens Account Transfer API v1.0.6
generated: '2026-09-05'
method: generated
source: openapi/_original/citizens-account-transfer-v1.json (harvested 2026-09-05 from
  https://developer.citizensbank.com/product/commercial-banking/api/accounttransfer-v1) plus
  https://developer.citizensbank.com/content/qut/CitizensAccountTransferAPIUserGuide.pdf v1.0 (2026-05-29).
operations:
  - initiateTransfer
base_url: https://apis.citizensbank.com/v1/account-transfer
---

# Transfer between a client's own Citizens accounts

Use this instead of the Payments API when both sides of the movement are accounts the same client
holds at Citizens. Payments is for external disbursement; this is internal liquidity movement.

**This flow moves money and Citizens publishes no reversal, no cancel and no undo window.**

## What the API will and will not do

- Near real-time, **same-day only**. Future-dated transfers are explicitly not supported in this version.
- **One transfer per call.** Batch transfers are not supported.
- No scheduling, no recurring transfer, no reversal operation.

## The call

`initiateTransfer` → `POST /initiate`

Body (`AccountTransferRequest`) requires `fromAccountNumber`, `toAccountNumber` and `amount`;
`memo` is optional. Minimum amount is $0.01.

Headers: `Authorization: Bearer <token>` from
`POST https://apis.citizensbank.com/as/token.oauth2` (client_credentials, `private_key_jwt` over
mTLS), plus `x-fapi-trace-id` (UUID) and the portal-issued `X-IBM-Client-Id`.

The response (`AccountTransferResponse`) carries `message`, `description`, `requestId`,
`requestTime`, `data` and `status`. Log `requestId` — it is the only correlation handle Citizens
gives you here, and there is no status-query operation on this API to look a transfer up again
afterwards.

## Idempotency: there is none on this operation

Unlike the Payments API — which at least rejects a replayed `paymentId` with `PMT1003` — Account
Transfer publishes **no duplicate-key protection at all**. There is no `Idempotency-Key` header and
no client-assigned transfer id in the request body.

The contract does declare a conflict code (`AT-409`), but Citizens does not document what triggers
it, so you cannot rely on it to catch a double-fire.

**Practical consequence for an agent:** a timeout on this call is genuinely ambiguous. You cannot
retry safely and you cannot query the result. Treat a network failure as *unknown*, surface it to a
human, and reconcile through Information Reporting balances or Citizens Client Services
(877-550-5933, 24x7) rather than by resending.

## Errors

`400`, `401` and `500` are declared, each returning the proprietary envelope
`{ result, source, requestId, errorDetails[] }`. `AT-429` is the rate-limit code — *"The user has
sent too many requests in a given time frame"* — and no `RateLimit-*` or `Retry-After` response
header is published, so back off on the status code alone. See
`errors/citizens-financial-group-error-codes.yml` and `rate-limits/`.
