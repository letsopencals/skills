# Resource-booking (courts, lanes, rooms, equipment) + group sessions

How to model businesses that book **things** (often many interchangeable units)
and/or **group** sessions, on top of the Opencals API. The padel & squash club
template (`template-volt`) is the worked example: court products (rentals) plus
coach-led individual and group trainings, over one store.

For the exact API/SDK shapes referenced here (custom duration, availability grid,
attendees), see the `opencals-storefront-api` skill's `custom-duration.md`,
`availability.md` and `guests-attendees.md`.

## The three conflict rules — pick the one that fits

Availability blocking is decided by one rule per booking, in priority order:

| Priority | Rule | Applies when | What blocks the slot | Use for |
|---|---|---|---|---|
| 1 | **Staff** | booking has a `staffMemberId` | that person's appointments across *any* product | coaches, stylists, clinicians, instructors |
| 2 | **Product Pool** | product has a `productPoolId`, no staff | any appointment for *any product in the pool* | one shared resource fronted by several products |
| 3 | **Self** | no pool, no staff | only that exact product's own appointments (same location) | an independent bookable unit (a court) |

### Courts / lanes / rooms → Self rule, one product per unit

Each unit is its **own Product** with **no staff** and **no pool**. Under the
Self rule it's an independent resource — booking it blocks only its own slots, so
all units book in parallel. Set `maxAttendees: 1` for exclusive use.

Group the units for display as **variants of one product group** (share
`productGroupId`) so the catalogue shows one "Padel Court" item whose `variants`
are the individual courts — each still a distinct `productId`/resource. Put them
in a **collection** ("Padel Courts") to drive a sport/category toggle.

### Variable duration → `allowCustomDuration`, NOT duration variants

A court that can be booked for 60 or 90 minutes is **one product** with:

- `allowCustomDuration: true`
- base `duration` = the pricing/booking unit in seconds (e.g. `1800` = 30 min)
- `maxDuration` = longest allowed booking in seconds (e.g. `5400` = 90 min), or `-1`

The customer books a positive integer multiple of the base; the backend prices it
as `ceil(bookedSeconds / baseSeconds) × price` and validates the multiple. So a
30-min base at €15 charges €30 for 60 min and €45 for 90 min.

> **Why not duration variants?** Two variant products (a "60 min" and a "90 min")
> are two *different* `productId`s. Under the Self rule each is its own resource,
> so booking the 60-min variant would NOT block the 90-min one on the same court.
> The court would double-book. `allowCustomDuration` keeps a court as ONE product
> (one resource) while still offering multiple priced durations. (A per-court
> ProductPool also works but is unnecessary given native custom duration.)

### Staff-led classes & lessons → Staff rule (+ group capacity)

Lessons and classes are ordinary products **assigned to coach staff** (`staffIds`).
The Staff rule stops a coach being double-booked across an individual lesson and a
group class at the same time.

- **Individual (1-on-1)**: `maxAttendees: 1`.
- **Group**: `maxAttendees > 1` (e.g. 4–6). Multiple customers book the same slot
  until full; pass `numberOfAttendees` on the booking. Each availability slot
  carries `attendees` and `maxAttendees` — render "X spots left" from those.
- Express levels (Beginner/Intermediate/Advanced) as variants under a per-format
  `productGroupId`, or as separate products in a "Training" collection.

## Grid data strategy (units × time)

`ProductService.getCurrentAvailabilities` returns slots for **one product**. To
paint a Playtomic-style courts × time grid:

1. Load the sport's court product group → its `variants` are the courts (rows).
2. **Fan out**: one availability call per court variant for the selected date at
   the base granularity (cache each under its own key; wrap in the rate-limit
   retry helper — see the core skill's `rate-limiting.md`).
3. A cell is *available* iff a base slot for that court starts at that time.
4. On cell click, offer durations (60/90). For a longer duration, re-query that
   court with `duration=<seconds>` (cached per court+date+duration) to confirm the
   full block fits, then book that court product for the returned slot.

Booking, cart and checkout are unchanged from `booking-flow.md` — the slot you
pass to `AppointmentService.create` simply carries the custom `from`/`to` span and
(for lessons) the chosen `staffMemberId`.

## Seeding notes

If you seed a resource-booking store, the seed layer supports it directly:
`ProductSeedConfig` has `allowCustomDuration?` / `maxDuration?` / `maxAttendees?`
/ `staffIds?`; single-venue businesses set `skipGeneratedLocations: true` so the
store isn't given auto-generated city branches. Demo bookings for no-staff
resource products are location-only appointments (staff null).
