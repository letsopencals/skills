# Rentals and multi-day bookings (cars, equipment, villas, boats)

How to model things that are booked **by the day** (or several days in a row)
on the Opencals API. The NOIR Drive car-rental template (`template-noir`) is the
worked example: twelve cars booked by the day with delivery, per-day extras and
a deposit at handover, plus chauffeur packages over one store.

For exact SDK shapes see the `opencals-storefront-api` skill:
`custom-duration.md`, `availability.md`, `guests-attendees.md`, `checkout.md`.
For the conflict rules (Self / Product Pool / Staff) see `resource-booking.md`.

## 1. Model: one unit = one day-based product

Each rentable unit (a car, a jet ski, a camera kit) is **its own product**:

| Field | Value | Why |
|---|---|---|
| `duration` | `86400` | One day is the pricing and booking unit |
| `allowCustomDuration` | `true` | The customer books N days = N base units |
| `maxDuration` | e.g. `2592000` (30 days), or `-1` | Longest rental |
| `maxAttendees` | `1` | Exclusive use |
| staff | none | No staff to block. The unit's own product pool (below) is the resource |
| product pool | one per unit | A booking blocks the unit at **every** location (see §6) |

Price is `ceil(bookedSeconds / 86400) × price`, so 3 days at AED 4,500 is
AED 13,500. Don't invent weekend or weekly rates in the frontend; if you need
them, model them in the store (e.g. discounts), not as client-side maths.

**One product per unit, no variants.** Variants are separate `productId`s and,
under the Self rule, separate resources: two variants of "Cullinan" (say
"self-drive" and "with delivery") would **not** block each other and the same
car would double-book. Give each car its own product (its own product group),
and express options as add-ons or locations instead.

## 2. Schedule: continuous, or none

The product must be available **across midnight**, otherwise availability stops
at the end of each day and nothing longer than a day can fit.

- Use a **continuous** schedule: every day 00:00–23:59:59 (24/7), or
- attach **no schedule** if the unit is always available.

Bookings longer than 24 hours are validated server-side against continuous
availability: the backend checks that the whole booked span sits inside one
merged free window (fixed in Oct 2026; earlier builds rejected multi-day
bookings on day-windowed slot generation).

Opening hours for handover (e.g. "we deliver 08:00–22:00") are **not** the
schedule. Keep the schedule 24/7 and offer handover windows in the UI (see 5).

## 3. Availability: ranges, then fit client-side

Slots don't work for multi-day UIs: `getCurrentAvailabilities` is per day and
a range picker needs to know whether *pick-up → return* is free as a whole.
Use the merged ranges instead:

```ts
// Server route (template: app/api/products/[slug]/ranges/route.ts)
const { data: ranges } = await ProductService.getCurrentAvailabilitiesMerged({
  path: { productId },
  query: { timezone: 'Asia/Dubai', locationId }, // NO duration
  throwOnError: true,
});
// ranges: CurrentAvailabilitySlot[] in UTC; on a 24/7 schedule a single range
// can span weeks, cut only by existing bookings.
```

- Call it **without `duration`**. With the 1-day base the response already
  drops windows shorter than a day; the client decides whether a longer span
  fits. Passing `duration` for every candidate length multiplies requests.
- Merge touching ranges before checking (tolerate a 1-second seam such as
  `23:59:59 → 00:00:00`).
- A span fits iff `[from, until)` lies entirely inside **one** merged range.

```ts
// lib/rental.ts in template-noir
function fitsRange(ranges, from, until) {
  const start = toMs(from), end = toMs(until);
  return mergeRanges(ranges).some((r) => r.start <= start && r.end >= end);
}
```

Use the same ranges to strike out unavailable days in the calendar
(`isDayAvailable`) and to cap the return date from a chosen pick-up date
(`latestReturnDate`). The server re-validates on booking, so handle a rejection
(someone else took the dates) by refreshing the ranges and asking again.

## 4. Slots are 00:00 local → 00:00 local, sent in UTC

A rental runs from the pick-up date at 00:00 to the return date at 00:00 in the
**store timezone** (not the browser's). Convert both to UTC for the slot:

```ts
import moment from 'moment-timezone';

function toAppointmentSlot(fromDate: string, untilDate: string, tz: string) {
  const from = moment.tz(fromDate, 'YYYY-MM-DD', true, tz).utc();
  const until = moment.tz(untilDate, 'YYYY-MM-DD', true, tz).utc();
  return {
    fromDate: from.format('YYYY-MM-DD'), fromTime: from.format('HH:mm:ss'),
    toDate: until.format('YYYY-MM-DD'), toTime: until.format('HH:mm:ss'),
  };
}

// Asia/Dubai (UTC+4), pick-up 10 Nov, return 13 Nov (3 days):
toAppointmentSlot('2026-11-10', '2026-11-13', 'Asia/Dubai');
// { fromDate: '2026-11-09', fromTime: '20:00:00', toDate: '2026-11-12', toTime: '20:00:00' }

await AppointmentService.create({
  body: {
    cartId,
    slot: { productId, ...slot, staffMemberId: null, locationId },
    numberOfAttendees: 1,
    customAttributes: { handover_time: '10:00–12:00', return_time: '10:00–12:00' },
    address, // only stored when locationId is a DELIVERY location
  },
});
```

Whole-day alignment keeps the ranges clean (bookings end exactly where the next
one may start), makes N days = N units exact, and matches rental norms
("return by the same time"). Show rentals as dates plus a day count, never as
clock times.

## 5. Handover details travel as custom attributes

The handover window, return window, a different collection address and a flight
number are **not** part of the slot. Send them as appointment
`customAttributes` (string → string, max 250 entries, keys up to 255 chars,
values up to 5,000 chars). They show up on the appointment in the dashboard and
in the customer's appointment responses, so the account page can display them.

NOIR Drive's keys: `handover_time`, `return_time`, `collect_address`,
`flight_number`, and `preferred_car` (chauffeur bookings). Keep them in one
config object so the booking flow and the account page read the same names.

## 6. Delivery vs collection: two locations

Attach each unit to two locations:

- a **PHYSICAL** location (the garage / shop) for "collect it yourself";
- a **DELIVERY** location for "bring it to me". The customer's address is passed
  as `address` on the appointment (or captured at checkout through the
  delivery-address flow) and is stored only for delivery-type locations.

Map "Deliver to / Collect from" in the UI to the `locationId` you put on the slot.

**Put each unit in its own product pool.** Without a pool a staffless product
uses the Self rule, which counts capacity **per location**. The garage and the
delivery location would then each sell the same car for the same days. A pool
is location-agnostic, so a booking at either location blocks the car at both.
With a pool, `getCurrentAvailabilitiesMerged` returns one set of ranges covering
both locations (each range lists both in `locationIds`).

## 7. Deposit: terms, not a charge

Opencals charges the booking total; it doesn't place card holds. Show the
security deposit per unit and make it part of the terms:

- a required **CHECKBOX** checkout question ("I accept the rental terms and the
  deposit hold at handover"), and
- the deposit amount on the car page, the summary and the confirmation.

The merchant takes the deposit at handover. Required document uploads (driving
licence, passport / ID) are **FILE_UPLOAD** checkout questions; the driver's date
of birth is a **SINGLE_LINE** question.

## 8. Per-day extras: `durationMultiplied` add-ons

Add-ons with `durationMultiplied: true` are charged per booked unit of the
parent appointment, so with a 1-day base they're exactly per day (an excess
waiver at AED 350/day × 3 days = AED 1,050). Fixed extras (child seat, intercity
delivery, prepaid fuel) leave `durationMultiplied` off and are charged once.

## 9. Rescheduling a rental

Keep the same number of days and move the pick-up date. The ranges endpoint has
**no `excludeAppointmentId`**, so add the booking's own `[from, to]` interval
back into the ranges on the client before running `fitsDates`; otherwise the
rental blocks itself. Then call `AppointmentService.reschedule` with the new
00:00-aligned slot. The server excludes the appointment itself when validating
and keeps the delivery address if you don't send a new one.

## 10. Staff-led add-on services (chauffeurs, instructors)

A chauffeur, skipper or instructor is a separate product **with staff** (Staff
rule) using the classic slot flow (`getCurrentAvailabilities` + time slots). If
the customer asks for a specific car with the driver, store it as a custom
attribute (`preferred_car`): that does **not** block the car's own availability.
If the car must be blocked too, book the car product as a second appointment in
the same cart.

## Checklist

- [ ] Each unit is its own product: `duration: 86400`, `allowCustomDuration`, `maxDuration`, no staff, `maxAttendees: 1`
- [ ] Continuous 24/7 schedule (or none)
- [ ] UI reads `getCurrentAvailabilitiesMerged` without `duration` and fits the span client-side
- [ ] Slot = pick-up 00:00 → return 00:00 in the store timezone, converted to UTC
- [ ] Handover window, return window, flight and collection address as `customAttributes`
- [ ] PHYSICAL + DELIVERY locations; address passed for delivery
- [ ] Deposit as a required checkbox question, never charged online
- [ ] Per-day extras use `durationMultiplied`
- [ ] Reschedule keeps the day count and unions the booking's own interval into the ranges
