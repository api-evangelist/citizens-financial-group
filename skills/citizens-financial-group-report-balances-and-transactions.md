---
name: citizens-report-balances-and-transactions
description: Pull the authorized account list, then balances and transaction history, from the
  Citizens Information Reporting API - the read-only cash-position surface for commercial clients.
api: Citizens Information Reporting API v1.0.16
generated: '2026-09-05'
method: generated
source: openapi/_original/citizens-information-reporting-v1.json (harvested 2026-09-05 from
  https://developer.citizensbank.com/product/commercial-banking/api/informationreporting-v1) plus
  https://developer.citizensbank.com/content/qut/CitizensInformationReportingAPIUserGuide.pdf v1.6 (2026-07-22).
operations:
  - getAccountList
  - getAccountDetails
  - getTransactions
base_url: https://apis.citizensbank.com/v1/information-reporting
---

# Report balances and transactions

Read-only. Nothing in this skill moves money, so `reversibility`, `idempotency` and `dry_run` are
all `na` here. Scope is Savings and Checking accounts.

Headers on every call: `Authorization: Bearer <token>` (client_credentials + `private_key_jwt` over
mTLS), `x-fapi-trace-id` (**required**, UUID, max 36), `x-fapi-channel-id` (optional),
`X-IBM-Client-Id`.

## Step 1 — discover what you are authorized to see

`getAccountList` → `GET /list?pageOffset=<n>&pageLimit=<n>`

**Both query parameters are required** — there is no unpaginated form. The response
(`AccountListQueryResponse`) returns `accountList`, `totalRows`, `totalPages` and `rowsPerPage`;
walk `totalPages` rather than guessing when to stop.

This is the only paginated surface in the entire Citizens estate. Every other API publishes no
pagination convention at all.

## Step 2 — balances

`getAccountDetails` → `POST /balance/query`

The body is an **array** of `AccountDetailRequest` objects — `accountNumber` required,
`routingNumber` optional — so one call covers many accounts.

Watch for **`206` Partial Content**: this operation declares it, meaning some accounts in your
batch resolved and others did not. A `206` is not a success. Reconcile the returned set against
what you asked for before reporting a cash position; a silently short array is how an agent reports
a wrong total.

## Step 3 — transactions

`getTransactions` → `POST /transactions/query`

Body is an array of `AccountTransactionRequest`: `accountNumber` required, plus optional
`routingNumber`, `startRangeOfDate`, `endRangeOfDate`, `limit` and `nextKey`.

- Default page is **100 transaction records**; up to **2000** can be requested via `limit`.
- `nextKey` is the cursor. Carry it forward until it stops coming back.
- A `204` means no transactions matched the range — an empty result, not a failure.

## Errors and limits

`400`, `401`, `404` and `500` all return the proprietary envelope
`{ result, source, requestId, errorDetails[] }`; this API declares more failure modes than most of
the Citizens estate, so handle each rather than treating everything as a generic error.

`IR-429` is the rate-limit code, with no published `RateLimit-*` or `Retry-After` response headers.
Since transaction pulls are the operation most likely to loop, cap concurrency yourself — the API
gives you no runtime signal to react to. See `rate-limits/`.
