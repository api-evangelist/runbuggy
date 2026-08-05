---
name: runbuggy-quote-and-place-order
description: Price a vehicle transport with RunBuggy and then commit it, handling the 202 accepted-then-poll pattern correctly. Use when moving one or more vehicles between two addresses.
api: runbuggy:orders
generated: '2026-08-05'
method: generated
source: openapi/runbuggy-orders.json, https://docs.runbuggy.com/docs/shipping/ea2a0d46dfc8d-handling-202-s
operations:
  - quoteOrderUsingPOST
  - createOrderUsingPOST
  - getOrderUsingGET
  - getFullOrderWithIdUsingGET
---

# Quote and place a RunBuggy order

## Before you start

- `Authorization: Bearer {token}` on every request. The spec declares this header as an
  apiKey, so you supply the literal `Bearer ` prefix.
- Base URL: the published spec declares the staging host
  `https://ng-staging.runbuggy.com/staging/api`. RunBuggy publishes no specification for
  production — get the production base URL from your RunBuggy contact.
- **There is no idempotency key on this API.** A retried `POST /orders` can create a
  second real transport. Everything below is built to avoid a blind retry.

## Step 1 — quote first (`quoteOrderUsingPOST`)

`POST /orders/quote` with an `OrderQuoteRequest`. This is the safe, non-committing
operation: it returns an `OrderQuoteResponse` without creating anything. Always quote
before creating, both to show a price and to validate the addresses and vehicles cheaply.

Quoting also returns 202. Poll it the same way as step 3.

## Step 2 — create the order (`createOrderUsingPOST`)

`POST /orders` with an `OrderRequest`:

```json
{
  "type": "BASIC",
  "notes": "optional free text",
  "vehicleTransferOrders": [
    {
      "directions": {
        "pickup":  { "address": "<full geocodable street address>" },
        "dropoff": { "address": "<full geocodable street address>" }
      },
      "vehicle": { "vin": "<17-character VIN>" }
    }
  ]
}
```

One entry in `vehicleTransferOrders` per vehicle. Each becomes its own Vehicle Transfer
Order with its own id, status and fare.

Vehicle identification, per the Vehicle Requirements guide: send a VIN **or** send
`year`, `make` and `model`. If you send both, the VIN is ignored.

If you are ordering on behalf of another company, see
`runbuggy-order-for-another-company`.

## Step 3 — handle the 202 (this is the step people get wrong)

`POST /orders` returns **202 Accepted**, not 201. The order is not yet created. The
response carries a `location` header with the URI to poll.

```
if response.status == 202:
    poll response.headers["location"]
```

Poll that URI with `getOrderUsingGET` semantics until the returned object's `status` is:

- `created` — the order exists. Read it.
- `error` — read the `errors` array of `{code, message, field}` objects. Do not retry
  blindly; fix the named `field` first.

Bound the loop (RunBuggy's own reference implementation uses 10 attempts at 5-second
intervals) and treat exhaustion as unknown-state, not as failure. Re-query with
`findOrderPaginatedUsingGET` before ever creating again.

Reference implementation published by RunBuggy:
https://github.com/runbuggyinc/api-doc-src/blob/master/shippers/src/202-example.js

## Step 4 — read the created order

- `getOrderUsingGET` — `GET /orders/{id}` for the order itself.
- `getFullOrderWithIdUsingGET` — `GET /orders/{id}/full` for the expanded form,
  which inlines every Vehicle Transfer Order, the transportation orders, driver
  locations and fare items. Use this once the order is created and you need the per-
  vehicle ids.

## Errors

The three published codes are `INVALID_VEHICLE_TRANSFER_ORDER`, `INVALID_ADDRESS` and
`INVALID_VEHICLE`. Each carries a `field` naming what to fix. 401, 403 and 404 responses
have no schema — treat them by status code alone.

## Do not

- Do not retry `POST /orders` on timeout. There is no idempotency key. Poll or re-query.
- Do not treat 202 as created.
- Do not assume a token is read-only. The same token can cancel orders.
