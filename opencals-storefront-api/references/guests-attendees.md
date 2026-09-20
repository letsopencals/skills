# Attendees & guests

Two distinct concepts:

- **Attendees** = how many people occupy the booking (capacity accounting). Set
  via `numberOfAttendees` when creating the appointment. Used for group products.
- **Guests** = named people (by email) invited to an appointment and copied on
  its notifications. Managed via `addGuest` / `removeGuest`.

## Creating an appointment (`AppointmentService.create`)

```ts
import { AppointmentService } from '@opencals/storefront-sdk';

const { data: appointment } = await AppointmentService.create({
  body: {
    slot: {
      productId,
      fromDate, fromTime, toDate, toTime, // from a CurrentAvailabilitySlot (UTC)
      staffMemberId,   // optional — when the customer picked a person
      locationId,      // optional
    },
    cartId,                 // attach to the current cart
    numberOfAttendees: 2,   // optional, defaults to 1
    // guests, addOns, customer, checkoutQuestionAnswers, address, customAttributes all optional
  },
});
```

`CreateAppointment` body fields (all besides `slot` optional): `customer`,
`internalNote`, `address` (DELIVERY-location service address), `checkoutQuestionAnswers`,
`numberOfAttendees`, `cartId`, `addOns`, `guests`, `customAttributes` (string→string map, ≤250 entries).

## Group capacity

For group products (classes, group trainings), each availability slot reports
`attendees` (already booked) and `maxAttendees` (capacity). Render "X spots left"
as `maxAttendees - attendees`, and cap `numberOfAttendees` at the remaining
capacity. The backend rejects an over-capacity `numberOfAttendees`.

## Guests

Invite guests either at creation (`guests: [...]` on the body) or afterwards:

```ts
// Add a guest to an existing appointment
await AppointmentService.addGuest({
  path: { appointmentId },
  body: { email: 'guest@example.com', notify: true }, // notify defaults to true
});

// Remove a guest
await AppointmentService.removeGuest({
  path: { appointmentId, guestId },
});
```

`AddGuest` = `{ email: string; notify?: boolean }`. Guests receive the
appointment's notification emails; they do **not** consume attendee capacity —
use `numberOfAttendees` for capacity, `guests` for who's cc'd.
