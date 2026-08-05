---
name: runbuggy-track-vehicle-transfer
description: Track a vehicle in transit with RunBuggy — read the per-vehicle transfer order, its 18-status lifecycle and driver telemetry, either by polling or by subscribing to webhooks.
api: runbuggy:orders
generated: '2026-08-05'
method: generated
source: openapi/runbuggy-orders.json, https://docs.runbuggy.com/docs/shipping/991cb1cc6950c-vehicle-transfer-order-statuses, https://docs.runbuggy.com/docs/shipping/5cc9374300b99-webhooks
operations:
  - getOrderVehicleTransferOrdersUsingGET
  - getVehicleTransferOrderUsingGET
  - getExpandedUsingGET
  - findVehicleTransferOrdersPaginatedUsingGET
  - findVehicleTransferOrdersExpandedPaginatedUsingGET
  - createWebhook
  - testWebhook
---

# Track a RunBuggy vehicle transfer

An Order is the commercial unit. A **Vehicle Transfer Order** is one vehicle's leg of it,
and it is the only object that has a live status and emits events. Track the transfer
order, not the order.

## Find the transfer orders

- From an order you already have: `getOrderVehicleTransferOrdersUsingGET` —
  `GET /orders/{id}/vehicle-transfer-orders`.
- Across your account: `findVehicleTransferOrdersPaginatedUsingGET` —
  `GET /vehicle-transfer-orders?page=0&size=25&sort=created.date,desc`.

Pagination is page-number style. The response envelope puts rows in `content` and
totals in `totalElements` / `totalPages`.

## Read one transfer

- `getVehicleTransferOrderUsingGET` — `GET /vehicle-transfer-orders/{id}` for the base
  object: `status`, `vehicle`, `directions`, `fare`, `transporter`, `orderId`.
- `getExpandedUsingGET` — `GET /vehicle-transfer-orders/{id}/expanded` when you need the
  live picture: `geolocation`, `timeline` (timestamped `Event` entries), `states`,
  `inspections`, and the parent `order` inlined.
- `findVehicleTransferOrdersExpandedPaginatedUsingGET` —
  `GET /vehicle-transfer-orders/expanded` for the expanded form in bulk.

## The status lifecycle

18 published values, in rough order:

`DRAFT` → `READY` → `AVAILABLE` → `CLAIMED` → `ASSIGNED` → `ACCEPTED` →
`DRIVER_ARRIVED` → `PICKED_UP` → `SIGNATURE_ON_PICKUP` → `PROVIDED_ETA_DROPOFF` →
`UNLOADED` → `DELIVERED` → `COMPLETE`

Off-path values: `UNCLAIMED` (a transporter dropped it), `APPROVED` (RunBuggy reviewed
after an unclaim), `REJECTED` (a driver declined), `CANCELED`, `ERROR`.

Reference: https://docs.runbuggy.com/docs/shipping/991cb1cc6950c-vehicle-transfer-order-statuses

Terminal for a happy path is `COMPLETE`, not `DELIVERED` — `DELIVERED` means the
paperwork was signed, `COMPLETE` means the delivery process finished.

## Prefer webhooks over polling

No rate-limit headers are published, and the platform enforces per-user request limits
that are not documented. Subscribe rather than poll.

1. `createWebhook` — `POST /webhooks` with the URL RunBuggy should POST to. You may
   supply a value RunBuggy will echo in the `Authorization` header of each delivery.
2. `testWebhook` — `POST /webhooks/{id}/test` to fire a real delivery and confirm your
   endpoint before going live. Do this every time.

The only published event type is `vehicleTransferOrder.updated`. The payload is
`{type, created, object}` where `object` is the expanded Vehicle Transfer Order.

**There is no payload signature** — no HMAC, no timestamp, no replay protection. Verify
what you can with the echoed `Authorization` value, and treat the event as a signal to
re-read the resource rather than as authoritative state.

## Careful

- The status vocabulary used by the embeddable status iframe (`VEHICLES_PICKEDUP`,
  `COMPLETED`, …) is a *different* enum from the API's (`PICKED_UP`, `COMPLETE`, …).
  Do not map one onto the other by string equality.
