# Resource-booking (courts, lanes, rooms, equipment) + group sessions

How to model businesses that book **things** (often many interchangeable units)
and/or **group** sessions, on top of the Opencals API. The padel & squash club
template (`template-volt`) is the worked example: four clubs (store locations),
each with its own padel doubles, padel singles and squash courts (rentals), plus
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

### Several venues → one location per club, courts assigned to one club

For a network of clubs, each club is a **store location** (with `latitude` /
`longitude` for "Near me") and each court is assigned to exactly **one**
location. Use one product group **per sport per club** (`Padel Court · Madrid`
with variants `Court 1`…) so a product page never mixes cities. Collections stay
per sport and span every club.

- Collection products don't carry locations. To know which club a court belongs
  to, read `ProductService.list` (each variant has `locations` and `collections`;
  the sport is the collection slug), or filter with `locationId`.
- Encode small per-unit facts in the variant title when the API has no field for
  them: VOLT uses `Court 7 · Singles` for a singles court (narrow tile, 2
  players); anything else is doubles.
- Start the booking page from **one club** (the linked one, the one remembered
  in the browser, or the nearest), never from every club at once.

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

### Shared floors sold by the seat → one capacity product per location

A coworking flex floor, a hot-desk area or an open studio has many identical
seats nobody picks individually. Model it as **one staffless product per
location with `maxAttendees` = the seats**, and book several seats with
`numberOfAttendees`. No staff and no pool, so the Self rule counts seats per
product and location; the slot stays open until the seats are taken.

Rooms in the same building stay one product each (`maxAttendees: 1`). For a
meeting room, `allowGuests` + `maxGuests` = seats − 1 lets the booker invite
the attendees (they get the calendar invite). `template-gable` is the worked
example: six houses, a flex floor per house, studio desks, rooms, booths and
studios, one product group per kind per house.

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

## Availability strategy for a club (map, grid, "free from")

`ProductService.getCurrentAvailabilities` answers for **one product and one
date**. Fanning that out per court per day from the browser burns the rate limit
fast. Instead, per club:

1. On the server, call `ProductService.getCurrentAvailabilitiesMerged` **once per
   court with no `date`**: it returns the court's free ranges for the whole
   booking horizon. Pass `locationId` and the club timezone, `throwOnError: true`,
   limited concurrency (2–3) and retry on 429.
2. Cache each court's ranges briefly (VOLT: `unstable_cache`, 60 s, tags
   `court:<id>` and `availability`), and drop the tag after a booking, cancel or
   reschedule so the change shows at once.
3. Return one response per club, e.g. `{ ranges: { [courtId]: [startMin, lenMin][] } }`.
4. Compute everything on the client from those spans: a court is free for a game
   at `t` when one span covers `[t, t + length]`; the grid cell, the hall-map
   tile state, "free from 21:00", the free-courts-per-half-hour pulse and the
   sold-out days on the day strip all come from the same maths. Switching day,
   length, sport or view makes no API calls.

The same route works for capacity products. Each merged range also carries
`attendees` and `maxAttendees`: adjacent segments that aren't full are merged and
`attendees` is the **peak** over the merged span, while full segments are left
out. So `maxAttendees − attendees` is a safe "seats left" for any window inside
the range (it may undercount, never overcount). Gable packs
`[startMin, lenMin, attendees, maxAttendees]` and returns one response per
**house** for every room, booth, desk and flex floor in it.

Book the chosen span as one slot (`from`/`to` covering the game) on that court
product; the backend re-checks it. If "any court" was picked, try the next free
court when the first one has just gone.

Booking, cart and checkout are unchanged from `booking-flow.md` — the slot you
pass to `AppointmentService.create` simply carries the custom `from`/`to` span and
(for lessons) the chosen `staffMemberId`.
