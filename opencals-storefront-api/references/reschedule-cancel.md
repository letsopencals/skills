# Reschedule & cancel

Both operate on an existing appointment id and are **gated by the service's
`rescheduleGap` / `cancelGap`** (minimum notice, in the product/service config).
A request inside the gap is rejected — read those values (they're on the
product/appointment payload) and hide/disable the action when the appointment is
too close, so the customer gets a clear message instead of a raw 4xx.

## Reschedule

Fetch new availability, then move the appointment.

```ts
import { ProductService, AppointmentService } from '@opencals/storefront-sdk';

// 1. Availability for the new day — EXCLUDE the appointment being moved,
//    otherwise its own current slot shows as busy and can't be re-picked.
const { data: slots } = await ProductService.getCurrentAvailabilities({
  path: { productId },
  query: { date, timezone, locationId, staffMemberId, excludeAppointmentId: appointmentId },
});

// 2. Move it.
await AppointmentService.reschedule({
  path: { appointmentId },
  body: {
    slot: { productId, fromDate, fromTime, toDate, toTime, staffMemberId, locationId },
    notifyCustomer: true,   // optional, defaults true
    // address?: Address     // only for DELIVERY-location appointments
  },
});
```

`RescheduleAppointment` = `{ slot: AppointmentDateRangeSlot; notifyCustomer?: boolean; address?: Address }`.

### Keep the length and the product

A customer reschedule moves the appointment; it can't change what was bought.

- The new slot must cover the **same number of billed units** as the current
  booking (a 2-hour booking of a per-hour product stays 2 hours; a 3-night stay
  stays 3 nights). A different length is rejected with `400` — a longer or
  shorter booking changes the price, and only the merchant can settle that
  (they edit the order in the dashboard).
- `slot.productId` must be the appointment's own product; anything else is
  ignored.

For custom-duration products, don't offer a free end-time picker. Fix the
length to the current booking's and pass it as `duration` when loading
availability, so only same-length slots are offered:

```ts
const duration = String((new Date(appointment.to).getTime() - new Date(appointment.from).getTime()) / 1000);
const { data: slots } = await ProductService.getCurrentAvailabilities({
  path: { productId },
  query: { date, timezone, duration, excludeAppointmentId: appointmentId },
});
```

### `excludeAppointmentId` availability note

`excludeAppointmentId` is supported on `GET .../current-availabilities`. It
removes the named appointment from the busy overlay so the customer can keep or
shift their own slot. Requires a recent SDK build — if your generated
`ProductGetCurrentAvailabilitiesData.query` doesn't include it, regenerate the
SDK against a current API spec.

## Cancel

```ts
await AppointmentService.cancel({
  path: { appointmentId },
  body: { notifyCustomer: true }, // optional, defaults true
});
```

`CancelAppointment` = `{ notifyCustomer?: boolean }`. Cancellation/refund policy
is enforced by the API; don't compute refunds client-side.

Reference implementation: the official templates'
`app/api/account/appointments/[appointmentId]/{reschedule,cancel}` routes, and
the UI in `template-frisor`'s `components/account/appointment-detail/`
(`reschedule-modal.tsx`, `cancel-modal.tsx`).
