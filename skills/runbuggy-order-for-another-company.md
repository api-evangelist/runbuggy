---
name: runbuggy-order-for-another-company
description: Place a RunBuggy order billed to another company you have been authorized to act for — resolve the authorized company, then set the payer on every vehicle.
api: runbuggy:companies
generated: '2026-08-05'
method: generated
source: openapi/runbuggy-companies.json, openapi/runbuggy-orders.json, https://docs.runbuggy.com/docs/shipping/94fced2e96c5f-placing-an-order-for-another-company
operations:
  - getCompaniesThatAuthorizedCompanyUsingGET
  - getCompaniesThatAuthorizedCompanyIdByUserIdUsingGET
  - createOrderUsingPOST
---

# Place a RunBuggy order for another company

## Precondition you cannot set yourself

The authorization is granted **out of band**. RunBuggy's guide is explicit: "Your
Runbuggy support contact can work with you to establish this authorization." There is no
API that grants it. The Companies API only *reads* authorizations that already exist —
if the company you want is not in the list, the answer is a support conversation, not a
different API call.

## Step 1 — resolve the authorized company

Either:

- `getCompaniesThatAuthorizedCompanyUsingGET` —
  `GET /companies/authorized/companies` — every company you are authorized for.
- `getCompaniesThatAuthorizedCompanyIdByUserIdUsingGET` —
  `GET /companies/authorized/companies/findByUserName` — narrow by username when you
  already know who you are acting for.

Note the base host differs from the Orders API: the Companies definition declares
`https://apps.runbuggy.com/staging-v2/api`, while Orders declares
`https://ng-staging.runbuggy.com/staging/api`. Same bearer token, different host.

You need the company `id` — a UUID.

## Step 2 — create the order with that payer

`createOrderUsingPOST` — `POST /orders`. Set `payer` inside **each** entry of
`vehicleTransferOrders`:

```json
{
  "notes": "These are the orders for Ed's lot.",
  "type": "BASIC",
  "vehicleTransferOrders": [
    {
      "directions": {
        "pickup":  { "address": "4884 E Butler Ave, Fresno, CA 93727" },
        "dropoff": { "address": "10050 N Metro Pkwy E, Phoenix, AZ 85051" }
      },
      "vehicle": { "vin": "KL4CJHSB3EB688122" },
      "payer": {
        "id": "7b34ba14-1f7a-4492-9fd7-4bef02ad6256",
        "type": "DEALER",
        "name": "Acme Auto World"
      }
    }
  ]
}
```

**Specify the payer for every vehicle.** It is a per-transfer-order field, not an
order-level one. A multi-vehicle order with the payer set on only the first entry will
bill the rest to you.

The example values above are RunBuggy's own, taken verbatim from their guide.

## Then

Handle the 202 exactly as in `runbuggy-quote-and-place-order` — this operation is
asynchronous and has no idempotency key.

Reference: https://docs.runbuggy.com/docs/shipping/94fced2e96c5f-placing-an-order-for-another-company
