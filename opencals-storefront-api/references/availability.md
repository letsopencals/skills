# Availability

Two endpoints, one per product. Both take `productId` in `path`; everything else
in `query`. **All returned dates/times are UTC.**

## `getCurrentAvailabilities` — exact bookable slots for a date

Use this to show the time slots a customer can actually book on a given day.

```ts
import { ProductService } from '@opencals/storefront-sdk';

const { data: slots } = await ProductService.getCurrentAvailabilities({
  path: { productId },
  query: {
    date,                    // required, 'YYYY-MM-DD'
    timezone,                // optional (e.g. 'Europe/London') — aligns slots to the local day
    locationId,              // optional
    staffMemberId,           // optional
    duration,                // optional, seconds AS A STRING (see below)
    excludeAppointmentId,    // optional, for rescheduling (see reschedule-cancel.md)
  },
});
```

Returns `CurrentAvailabilitySlot[]`:

```ts
type CurrentAvailabilitySlot = {
  productId: string;
  fromDate: string;   // 'YYYY-MM-DD' (UTC)
  fromTime: string;   // 'HH:MM:SS' (UTC)
  toDate: string;     // 'YYYY-MM-DD' (UTC)
  toTime: string;     // 'HH:MM:SS' (UTC)
  // + attendees / maxAttendees for group products, staffMemberIds[] / locationIds[]
};
```

## `getCurrentAvailabilitiesMerged` — broad ranges for calendar highlighting

Returns merged availability **windows** across the horizon (not individual
slots). Use it to highlight which dates have any availability, then call
`getCurrentAvailabilities` for the day the customer picks. `date` is optional —
omit it to get the full horizon.

```ts
const { data: ranges } = await ProductService.getCurrentAvailabilitiesMerged({
  path: { productId },
  query: { locationId, staffMemberId, timezone, duration }, // all optional; date optional too
});
```

Endpoints:
- `GET /storefront/products/{productId}/current-availabilities`
- `GET /storefront/products/{productId}/current-availability-ranges`

## The `duration` param (duration-aware slots)

`duration` is a **string of seconds**. When set, both endpoints return only the
windows/slots long enough to fit that booking length; when omitted, the product's
**base duration** is used. This is how a variable-length product (see
`custom-duration.md`) shows the right slots for a 60- vs 90-minute booking:

```ts
// 90-minute slots for a court that has a 30-minute base:
query: { date, timezone, duration: String(90 * 60) } // '5400'
```

Ranges shorter than the requested duration are dropped.

## `excludeAppointmentId` (rescheduling)

When fetching availability to **reschedule** an existing appointment, pass its id
as `excludeAppointmentId` so the slot it currently occupies isn't reported as
busy — otherwise the customer can't rebook their own time. Details and the full
reschedule flow: `reschedule-cancel.md`.

## Grid UIs (units × time) — fan out, don't invent a batch call

There is no multi-product availability endpoint. To paint a "courts × time" (or
"rooms × time") grid, **fan out** one `getCurrentAvailabilities` call per product
for the selected date and cache each under its own key (e.g. one SWR key per
product+date+duration). A cell is available iff that product has a slot starting
at that time. This is the padel/volt template's strategy — see the
`opencals-build-booking-site` skill's `references/resource-booking.md`.

Because fan-out multiplies requests, wrap these calls in the retry helper from
`rate-limiting.md`.
