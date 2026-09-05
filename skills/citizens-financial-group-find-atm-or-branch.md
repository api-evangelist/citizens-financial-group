---
name: citizens-find-atm-or-branch
description: Look up Citizens ATMs and branches by postal code, state and city, coordinates or
  routing number - the one Citizens surface that needs no OAuth, but is published only on sandbox.
api: Citizens ATM Locator API v1.0.10 and Citizens Branch Locator API v1.0.10
generated: '2026-09-05'
method: generated
source: openapi/_original/citizens-atm-locator-v1-sandbox.json and
  openapi/_original/citizens-branch-locator-v1-sandbox.json, harvested 2026-09-05 from
  https://sandboxdeveloper.citizensbank.com/product/127.
operations:
  - findAtmsByPostalCode
  - findBranchesByPostalCode
base_url: https://sandboxapi.citizensbank.com/v1/atm-locator
---

# Find a Citizens ATM or branch

Read-only reference data. No OAuth, no mTLS, no token exchange.

## Two things to know before you build on this

1. **These contracts are published only on the sandbox portal.** ATM Locator and Branch Locator do
   not appear in the production catalogue at `https://developer.citizensbank.com/api`, and the only
   host named in either contract is `sandboxapi.citizensbank.com`. Citizens publishes no production
   base URL for the locators. Do not assume one by swapping the hostname.
2. **The credential is different from the rest of the estate.** Both use a `Partner-ID` request
   header issued to registered vendors — *not* the client-credentials + mTLS flow every other
   Citizens API requires. If you already hold Citizens commercial-banking credentials, they do not
   work here.

## ATM Locator — `https://sandboxapi.citizensbank.com/v1/atm-locator`

- `findAtmsByPostalCode` → `GET /postalcode/{postalCode}` — the only operation Citizens names.
- `GET /state/{state}/city/{city}` — no operationId in the published contract.
- `GET /latitude/{latitude}/longitude/{longitude}` — see the defect note below.

Results carry the ATM location, hours of operation, and whether it is a standalone ATM or sits
inside another entity such as a supermarket.

## Branch Locator — `https://sandboxapi.citizensbank.com/v1/branch-locator`

- `findBranchesByPostalCode` → `GET /postalcode/{postalCode}` — the only named operation.
- `GET /state/{state}/city/{city}`
- `GET /routingnumber/{routingnumber}` — useful for resolving a Citizens routing number to the
  branch that owns it.
- `GET /latitude/{latitude}/longitude/{longitude}` — see below.

## A defect in the published contracts

Both contracts write the coordinate path as `/latitude/\{latitude}/longitude/{longitude}` — with a
literal backslash escaping the first brace. `latitude` is therefore not a valid path-template
parameter, and a generated client will put the backslash on the wire.

Build the coordinate lookup by hand if you need it. We record this as observed rather than
correcting it; the original contracts are never mutated. See
`overlays/citizens-financial-group-atm-locator-overlay.yaml`.

## Errors

Neither contract declares anything but `200`. There is no machine-readable failure mode here at
all — no 4xx, no 5xx, no error schema — so handle non-200 responses defensively and do not expect
the proprietary `{ result, source, errorDetails[] }` envelope the commercial-banking APIs return.
