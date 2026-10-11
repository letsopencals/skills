# Rentals and multi-day bookings (cars, equipment, villas, boats)

How to model things that are booked **by the day** (or several days in a row)
on the Opencals API. The NOIR Drive car-rental template (`template-noir`) is the
worked example: twelve cars booked by the day with delivery, per-day extras and
a deposit at handover, plus chauffeur packages over one store.

Villas, cabins and rooms booked **by the night**, with fixed check-in and
check-out times, follow the same model with one change to the slot: see §11.
The Bali Villa Rentals template (`template-bali-villas`) is the worked example
for that.

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

## 11. Nightly stays: fixed check-in and check-out

For villas, cabins, guest houses and rooms, keep everything above (one product
per unit, 1-day base, `allowCustomDuration`, continuous schedule, its own
product pool, ranges without `duration`) and add fixed times:

| Field | Value |
|---|---|
| `fixedTimes` | `true` |
| `fixedStartTime` | check-in, e.g. `"15:00:00"` |
| `fixedEndTime` | check-out, e.g. `"11:00:00"` |

**The slot runs check-in date at `fixedStartTime` → check-out date at
`fixedEndTime`**, local to the product's schedule timezone (the store timezone
if it has no schedule), sent in UTC. Not 00:00 → 00:00 as in §4.

```ts
// lib/stay.ts in template-bali-villas (Asia/Makassar, UTC+8)
const times = stayTimesFor(product); // { checkIn: '15:00', checkOut: '11:00' } from fixedStartTime/fixedEndTime
toStaySlot('2026-11-14', '2026-11-17', 'Asia/Makassar', times);
// { fromDate: '2026-11-14', fromTime: '07:00:00', toDate: '2026-11-17', toTime: '03:00:00' }  → 3 nights
```

- **The server enforces it.** Since October 2026, a customer booking
  (`POST /storefront/appointments`, with or without a cart) or a customer
  reschedule whose slot doesn't start at `fixedStartTime` and end at
  `fixedEndTime` in that timezone is rejected with `400`: "This product must be
  booked from 15:00 to 11:00 (Asia/Makassar time)". Build the slot on your
  server from the product's own fixed times (`stayTimesFor`), never from
  client-sent times. Merchant bookings in the dashboard are exempt, which is how
  an early check-in or a late check-out gets applied.
- **Pricing rounds up.** An N-night stay is `(N−1)·24h + 20h`, not a multiple of
  the base, so the "whole multiple of `duration`" check is skipped for these
  products (only `maxDuration` applies) and the API prices `ceil(booked / 86400)`
  = N nights. Show `nights × price` as a preview only; the cart total is the
  source of truth.
- **Night availability.** A night on date D is free if D at check-in → D+1 at
  check-out fits inside one merged range (`isNightAvailable`). A stay fits if
  check-in → check-out fits inside one range (`fitsStay`). `latestCheckout`
  caps the check-out date at the next booking (and at the longest stay).
- **Turnover days.** Because a night only blocks check-in → next day's
  check-out, a check-out day can be the next guest's check-in day (out at 11:00,
  in at 15:00). Don't mark the check-out day as booked in the calendar.
- **Minimum nights** aren't a product field. Keep them in site content (the
  template has `minNights` per villa in `lib/villa-content.ts`) and enforce
  them in the calendar.
- **Guest details** go in `customAttributes`, not the slot. Lumbung Sidemen's
  keys: `adults`, `children`, `infants`, `arrival_time`, `flight_number`,
  `occasion`.
- **Extras** like an airport transfer, a floating breakfast or an "early
  check-in request" are add-ons with `durationMultiplied` **off** (charged once
  per stay).
- **Changing dates.** Keep the same number of nights (the backend doesn't
  reprice an order on reschedule), add the stay's own interval back into the
  ranges as in §9, and rebuild the slot with the same fixed times.
- Set `cancelGap` (14 days in the demo) and quote the cutoff from it so the copy
  and the API agree.

## Checklist

- [ ] Each unit is its own product: `duration: 86400`, `allowCustomDuration`, `maxDuration`, no staff, `maxAttendees: 1`
- [ ] Continuous 24/7 schedule (or none)
- [ ] UI reads `getCurrentAvailabilitiesMerged` without `duration` and fits the span client-side
- [ ] Slot = pick-up 00:00 → return 00:00 in the store timezone, converted to UTC (day rentals), or check-in date at `fixedStartTime` → check-out date at `fixedEndTime` in the schedule timezone (nightly stays with `fixedTimes`; enforced server-side, `400` otherwise)
- [ ] Handover window, return window, flight and collection address as `customAttributes`
- [ ] PHYSICAL + DELIVERY locations; address passed for delivery
- [ ] Deposit as a required checkbox question, never charged online
- [ ] Per-day extras use `durationMultiplied`
- [ ] Reschedule keeps the day count and unions the booking's own interval into the ranges
- [ ] Nightly stays: turnover day bookable, minimum nights in site content, guest counts as `customAttributes`
