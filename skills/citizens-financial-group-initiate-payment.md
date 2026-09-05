---
name: citizens-initiate-payment
description: Send an RTP or ACH payment from a Citizens commercial account, check the counterparty
  can receive it first, and resolve the final outcome - which is only knowable by polling status.
api: Citizens Payments API v3.1.11
generated: '2026-09-05'
method: generated
source: openapi/_original/citizens-payments-v3.json (harvested 2026-09-05 from
  https://developer.citizensbank.com/product/commercial-banking/api/payments-v3) plus
  https://developer.citizensbank.com/content/qut/CitizensPaymentAPIUserGuide.pdf v1.3 (2025-12-11).
operations:
  - checkParticipantStatus
  - initiatePayment
  - retrievePaymentStatus
base_url: https://apis.citizensbank.com/v3/payments
---

# Initiate a Citizens payment

**This flow moves money and Citizens publishes no reversal.** There is no cancel, void or recall
operation anywhere in the contract. Read `conventions/citizens-financial-group-conventions.yml`
(`reversibility:`) before wiring an agent to it.

## Before you can call anything

Citizens is not self-serve. An Implementation Manager provisions source-IP allowlisting, collects
the mTLS public key and the JWKS URL, and only then issues portal credentials. All testing happens
in the sandbox first — this is mandatory, not optional. See `sandbox/`.

Every request carries:

- `Authorization: Bearer <token>` — from `POST https://apis.citizensbank.com/as/token.oauth2`,
  grant `client_credentials`, client authenticated with a `private_key_jwt` assertion over mTLS.
- `x-fapi-trace-id` — **required**, a UUID, max 36 chars, unique per request.
- `x-fapi-channel-id` — optional, max 20 chars.
- `X-IBM-Client-Id` — issued at portal onboarding.

## Step 1 — check the counterparty first (RTP only)

`checkParticipantStatus` → `POST /participant-status/query`

Body requires `routingNumber` and `paymentType`. The response tells you `inNetwork`, `available`,
`participantName` and an `eligibleServices` list drawn from the TCH RTP vocabulary
(`CREDIT_TRANSFER`, `REQUEST_FOR_PAYMENT`, `ACKNOWLEDGMENT`, `REMITTANCE`,
`REQUEST_FOR_INFORMATION`, `REQUEST_FOR_RETURN_OF_FUNDS`).

An unreachable receiving bank is the cheapest failure to catch here rather than after submission.
A `204` means no content came back for that routing number — treat it as "not established", not as
"in network".

## Step 2 — submit

`initiatePayment` → `POST /initiate-payment`

Required: `paymentId`, `paymentType`, `amount`, `paymentAccountInformation`
(`routingNumber` + `accountNumber`), `counterpartyAccountInformation` (`routingNumber` +
`accountNumber`). Then `rtpDetails` or `achDetails` depending on `paymentType`.

**`paymentId` is your duplicate guard and you assign it.** Max 15 characters, and it must be unique
per submission. A replay is rejected with `PMT1003` — *"paymentId should be unique. paymentId
provided in this request has already been processed."*

This is not idempotent replay. A retry returns a **400 error, not the original 200 response**. If
you lose the first response you must reconcile through step 3, never by resending. Coverage is
recorded as `partial` in `conventions/`, and this operation is the whole of that scope.

For ACH, `achDetails` requires `standardEntryClassCode` (Nacha SEC: `CCD`, `PPD`, `WEB`, `CTX`),
`companyIdentification`, `companyName`, `companyEntryDescription`, `effectiveEntryDate` and
`counterpartyInformation`. The SEC code changes the validation rules — `counterpartyName` is 16
chars for `CTX` and 22 for the others (`PMT1213`); `CCD`/`PPD`/`WEB` take one addenda record
(`PMT1221`), `CTX` up to 9999 (`PMT1222`).

A prenote — `amount` of 0 — validates an account without moving money (`PMT1223`). That is the
closest thing to a dry run on this surface.

The `200` gives you back `paymentId`, `paymentStatus` and `receivedDateTime`. **It is not the
outcome.**

## Step 3 — resolve the real outcome

`retrievePaymentStatus` → `POST /payment-status/query`, body `paymentId` + `paymentType`.

RTP statuses: `RECEIVED`, `IN_PROGRESS`, `COMPLETED`, `REJECTED`, `EXPORTED`, `ACKNOWLEDGED`.
A rejection carries `rejectCode` and `rejectDescription` — ISO 20022 `ExternalStatusReason1Code`
values coming back from the RTP network. The full catalogue is in
`errors/citizens-financial-group-decline-codes.yml`.

The accept/reject decision belongs to the receiving bank, so it can only ever arrive here. Do not
report success off the step-2 `200`.

## Errors

Every failure returns `application/json` with the proprietary envelope
`{ result: FATAL|WARNING, source, errorDetails[] { code, description, messageDetail } }` — not
RFC 9457 problem+json. `WARNING` means the call succeeded but the transaction is not absolutely
complete; do not treat it as a clean success. Codes are catalogued in
`errors/citizens-financial-group-error-codes.yml`.

`429` exhaustion is documented but Citizens publishes **no** `RateLimit-*` or `Retry-After` response
headers, so a client has no runtime signal — back off on the status code alone. See `rate-limits/`.
