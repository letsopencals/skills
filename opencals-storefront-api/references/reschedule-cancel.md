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

Reference UI: `frontend/apps/storefront/components/account/appointments/`
(`reschedule-appointment-modal.tsx`, `cancel-appointment-modal.tsx`).
