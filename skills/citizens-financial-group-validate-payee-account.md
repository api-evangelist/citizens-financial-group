---
name: citizens-validate-payee-account
description: Verify a payee's account number, routing number and beneficiary name against Citizens
  Account Validation before sending an irrevocable payment, and poll the reference id for the result.
api: Citizens Account Validation API v1.0.13
generated: '2026-09-05'
method: generated
source: openapi/_original/citizens-account-validation-v1.json (harvested 2026-09-05 from
  https://developer.citizensbank.com/product/commercial-banking/api/accountvalidation-v1) plus
  https://developer.citizensbank.com/content/qut/CitizensAccountValidationAPIUserGuide.pdf v1.6 (2026-04-30).
operations:
  - getAccountInquiry
  - getAccountInquiryByRefId
base_url: https://apis.citizensbank.com/v1/account-validation
---

# Validate a payee account before you pay it

**This call is billed.** Citizens publishes a unit price of $1.25 per successful validation — the
only published unit price anywhere in the Citizens API estate. See
`plans/citizens-financial-group-plans-pricing.yml`. An agent looping over a customer list is
spending real money per row.

## Headers

`Authorization: Bearer <token>` (client_credentials + `private_key_jwt` over mTLS),
`x-fapi-trace-id` (**required**, UUID, max 36), `x-fapi-channel-id` (optional), plus the
`X-IBM-Client-Id` from portal onboarding.

## Step 1 — submit the inquiry

`getAccountInquiry` → `POST /initiation`

Body is `AccountStatusInquiryRequest`: `accountIdentifier` and `bankIdentifier`. The response
(`AccountInquiryInitiateResponse`) returns `data.referenceIdentifier.referenceId`.

If the answer is available immediately, the same response also carries `inquiryStatus` with
`accountStatus`, `accountStatusCode`, `accountStatusMessage` and `nameMatchStatus` — check for it
before you queue a poll.

Bank identifier formats are validated per scheme and rejected with `AV-6001` through `AV-6010`:
SWIFT_ID 8 or 11 alphanumerics, IBAN 5–34, IFSC 11, CLABE 18, USABA 9, BRAZIL_BANK_CODE 3, CBU 22,
CVU 22, CCI 20, CACPA 9. Validate locally first — a malformed identifier still costs a round trip.

## Step 2 — read the result

`getAccountInquiryByRefId` → `GET /status?referenceId=<referenceId>`

**Two hard limits Citizens states in the user guide, and both bite agents:**

1. Once the inquiry is complete, the status can be requested **at most three times**. Exceeding it
   returns `AV-4001` — *"You have exceeded the rate limit for the number of status requests."*
   Cache the result on the first read. A retry loop will burn the budget in three passes.
2. If no status is received within **24 hours**, the request is **automatically cancelled** by
   Citizens. This is server-side; there is no client-invocable cancel. It is the only stated
   reversal window on the whole Citizens surface, and it applies to an inquiry, not to money.

## Interpreting the answer

- `accountStatus` — open, closed, unverified, incorrect, rejected, or open-credits-only.
- `nameMatchStatus` — partial, full, or no match.

A partial name match is not a pass. Citizens' own framing is that real-time payments are
irrevocable, which is exactly why this check exists: there is no reversal on `initiatePayment`, so
this is the last point at which a wrong counterparty is cheap to catch.

## Errors

Proprietary envelope `{ result, source, requestId, errorDetails[] }` on 400/401/500. Note the
contract declares no `404`. Codes are in `errors/citizens-financial-group-error-codes.yml`.
