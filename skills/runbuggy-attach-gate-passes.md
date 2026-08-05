---
name: runbuggy-attach-gate-passes
description: Attach gate passes (release paperwork) to RunBuggy vehicle transfer orders, using either the create-order-then-attach flow or the create-pass-then-order flow.
api: runbuggy:orders
generated: '2026-08-05'
method: generated
source: openapi/runbuggy-orders.json, https://docs.runbuggy.com/docs/shipping/39130c4ea9ccd-gate-passes
operations:
  - createGatePassUsingPOST
  - addGatePassUsingPOST
  - getGatePassesUsingGET
  - getGatePassUsingGET
  - createOrderUsingPOST
  - getFullOrderWithIdUsingGET
---

# Attach gate passes to a RunBuggy transfer

A gate pass is a file bound to a **Vehicle Transfer Order** — it is modelled as an
`Attachment`, not as its own entity. RunBuggy documents two flows and they are not
interchangeable.

## Flow A — order first, then attach

Use when the order is already placed.

1. `createOrderUsingPOST` — `POST /orders`.
2. **Wait for the order to fully materialise.** The 202 must resolve and every Vehicle
   Transfer Order must exist before you attach anything. See
   `runbuggy-quote-and-place-order` step 3.
3. `getFullOrderWithIdUsingGET` — `GET /orders/{id}/full` — the expanded order inlines
   every Vehicle Transfer Order, which is where you get the per-vehicle ids.
4. `addGatePassUsingPOST` — `POST /vehicle-transfer-orders/{id}/gate-passes` — once per
   transfer order that needs one.

Skipping step 2 is the failure mode: attaching before the transfer orders exist has
nothing to attach to.

## Flow B — pass first, then order

Use when a human is assembling an order and wants the paperwork set before submitting.

1. `createGatePassUsingPOST` — `POST /vehicle-transfer-orders/gate-passes`. This returns
   the resource path of the created pass.
2. `createOrderUsingPOST` — `POST /orders`, supplying the list of gate passes created in
   step 1 inside the relevant `vehicleTransferOrders` entries.

## Read them back

- `getGatePassesUsingGET` — `GET /vehicle-transfer-orders/{id}/gate-passes` — all passes
  on a transfer.
- `getGatePassUsingGET` — `GET /vehicle-transfer-orders/{id}/gate-passes/{passId}` — one
  pass.

## Notes

- No idempotency key. If an attach call times out, read back with
  `getGatePassesUsingGET` before retrying, or you will attach the file twice.
- Reference: https://docs.runbuggy.com/docs/shipping/39130c4ea9ccd-gate-passes
