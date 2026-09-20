# Custom duration

Some products can be booked for a variable length (a court for 60 or 90 minutes,
a studio by the hour). This is a **product configuration**, set in the dashboard,
not a client-side trick.

## Product config

- `allowCustomDuration: true`
- base `duration` = the pricing/booking unit, in **seconds** (e.g. `1800` = 30 min)
- `maxDuration` = longest allowed booking, in seconds (e.g. `5400` = 90 min), or `-1` for unlimited

The customer books a positive-integer **multiple** of the base unit. The backend
prices it as `ceil(bookedSeconds / baseSeconds) × price` and validates the
multiple.

Example: a 30-minute base at £15 → £30 for 60 minutes, £45 for 90 minutes.

## Client flow

1. Query availability for the chosen length by passing `duration` (seconds, as a
   string) to `getCurrentAvailabilities` — you get back only the slots long
   enough to fit. See `availability.md`.
2. Build the appointment slot spanning the full booked window (the returned
   slot's `fromTime`→`toTime` already reflects the requested duration).
3. Create the appointment with that slot (see `guests-attendees.md`). Pricing is
   derived server-side from the span — you don't send a price.

```ts
// 90-minute booking on a 30-min-base product
const { data: slots } = await ProductService.getCurrentAvailabilities({
  path: { productId },
  query: { date, timezone, duration: String(90 * 60) }, // '5400'
});
// slots[i] spans 90 minutes; pass one straight into the appointment slot.
```

## Do NOT model variable duration as separate products

Two "60 min" and "90 min" **variant products** are two different `productId`s. If
the product is a self-blocking resource (a court — see the resource-booking
reference), booking the 60-min variant would **not** block the 90-min one on the
same unit, and the court double-books. Keep it as ONE product with
`allowCustomDuration` so it stays a single resource while still offering multiple
priced lengths.

Duration-scaled add-ons (e.g. equipment charged per unit of time) scale their
quantity with the parent appointment's duration units automatically.
