# Schools and classes (languages, music, coding, tutoring)

How to model a school on the Opencals API: group classes with levels, private
lessons, a free trial, teachers, campuses and an online classroom. The
Fluently language-school template (`template-fluently`, seed dataset
`language_school`) is the worked example: six languages, two New York campuses
and a live online classroom over one store.

For exact SDK shapes see the `opencals-storefront-api` skill:
`products-variants.md`, `availability.md`, `guests-attendees.md`, `checkout.md`.
For the conflict rules (Self / Product Pool / Staff) see `resource-booking.md`.

## 1. Model

| Concept | Opencals | Notes |
|---|---|---|
| Subject (a language, an instrument) | **Collection** | One per subject. Keep a separate "start here" collection for the trial and drop-in events |
| Group class | **Product**, `maxAttendees > 1` | e.g. "Spanish · In-person group class", 90 min, 8 seats |
| Level (A1, A2, B1…) | **Variant** of the group class | Variant title carries the level code ("Beginner · A1") |
| Private lesson | **Product**, `maxAttendees: 1` | Variants are lengths (45 / 60 min) |
| Free trial | 1-seat product priced `0` | Checkout's no-payment-required path books it without a card |
| Teacher | **Staff member**, assigned per class (`staffIds`) | The Staff rule stops a teacher being double-booked across group and private lessons |
| Campus | `PHYSICAL` location | |
| Online classroom | `ONLINE` location with a `link` | The link is the meeting URL (see §4) |
| Extras (workbook, recording) | Add-ons | Product-specific add-ons for online-only extras |
| Placement info (current level, goals) | Checkout questions | |

**Derive the format from the data, never from slugs.** A group class whose
locations are all `ONLINE` is an online group; with any physical location it's
on campus. A 1-seat class is private, or a trial when it costs `0`. Subjects
come from collections and each teacher's subjects, campuses and online flag
come from the products and locations they're assigned to. Then the same site
works for any school without code changes.

**Colors:** give each product a dashboard `color` and map that to the UI
swatch. Don't key colors off subject slugs.

## 2. Levels as variants

Each level is its own variant (`productId`) with its own price and slug.
- Link straight to a level with `/booking/<variant-slug>`.
- Show level chips on the class card from `variants[].variantTitle`.
- A "level finder" quiz can recommend a code ("A2") and filter
  `/classes?level=A2`. It's pure frontend; nothing is stored.

Under the Staff rule, every level of a class is blocked by its teacher's other
bookings. Two teachers on the same class can each run a session at the same time.

## 3. Seats left and a timetable

Each availability slot carries `attendees` and `maxAttendees`; seats left is
`maxAttendees - attendees`, and a slot is full when they're equal. Booking more
than one seat uses the attendee count (see `guests-attendees.md`).

For a "this week" timetable across every group class:
- Call `getCurrentAvailabilitiesMerged` once per group **variant**, with no date.
  One call returns the whole horizon.
- Cap concurrency (about 6). The storefront API rate-limits per key + IP.
- Cache the result server-side for a few minutes (`unstable_cache`) so one
  fan-out serves every visitor.
- Collapse slots with the same start (one per teacher) into one row with the
  most free seats. Use `slot.staffMemberIds` to show or preselect the teacher.
- Deep-link each row to `/booking/<variant-slug>?date=YYYY-MM-DD&staff=<id>`.
  Validate the params against the variant before applying them.

## 4. Online lessons and the Join button

Set the meeting URL as the **Online location's `link`** in the dashboard. It
comes back on the booked appointment's `location.link` (appointment detail,
order detail, account lists). Show a **Join lesson** button for appointments at
an `ONLINE` location, enabled from about 15 minutes before `from` until `to`.
With no link, tell the student the teacher will send one.

Render lesson times in the store timezone on the server and switch to the
visitor's timezone after hydration, so online students abroad see local times
without a hydration mismatch.

## 5. Keep it honest

- Count stats (languages, teachers, campuses) from the API. Don't hard-code
  numbers like "50,000 students".
- Chat, shared materials and attendance aren't part of the storefront API.
  Don't fake them in the UI; label them as coming soon or leave them out.
