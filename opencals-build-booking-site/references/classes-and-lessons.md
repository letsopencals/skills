# Schools and classes (languages, music, coding, tutoring, fitness studios)

How to model a school on the Opencals API: group classes with levels, private
lessons, a free trial, teachers, campuses and an online classroom. The
Fluently language-school template (`template-fluently`, seed dataset
`language_school`) is the worked example: six languages, two New York campuses
and a live online classroom over one store. For a fitness or Pilates studio timetable
with beds left and Full classes, see §6 (the Tensile template).

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

## 6. Fitness and Pilates studios: the Tensile worked example

The Tensile reformer Pilates template (`template-reformer`, seed dataset
`reformer_studio`, demo https://template-reformer.vercel.app) applies the same
model to a studio timetable: three Berlin studios plus live online classes,
eight instructors, and every class capped at the beds in its room.

| Concept | Opencals | Notes |
|---|---|---|
| Class type (Reformer Flow, Foundations) | **Product** (a product group) | Its variants are the weekly series below |
| One weekly series | **Variant** per class × studio × instructor | Slug `flow-mitte-lena`. Own schedule with one window per weekly start, its studio as the **only** location, its instructor as the **only** staff member |
| Capacity | `maxAttendees` = beds in the room | Beds left = `maxAttendees - attendees` on each slot |
| Rooms and bed plans | Template content (`lib/site-config.ts`) | Rooms aren't API entities. Map class × studio → room for the SVG plan |
| Privates (1:1, duet) | 1-seat variants (`private-<studio>-solo`, `-duet`) sharing **one product pool per studio** | The pool is the private room: a 1:1 and a duet can't overlap. A duet is one booking for two, sent as `customAttributes.people = "2"` |
| Workshops | Dated products in a `workshops` collection | One-off; never inferred as Full |
| Online classes | `ONLINE` location with a `link` | Join button from 10 min before start (see §4) |
| Add-ons | Grip socks, towel hire, recovery shake | |
| Waiver, pregnancy | Checkout questions | The waiver is a required checkbox |
| Discounts | Automatic 30% first-order intro, automatic 20% weekday off-peak (class **start** Mon–Fri 10:00–16:00), code `TENSILE10` | They don't combine: the best one wins per line. If an automatic discount beats an entered code, the code comes back `codeStatus: 'applied'` but is absent from `appliedDiscounts`: say "a better offer is already applied" |

**Why one variant per class × studio × instructor:** every timetable row then
names its teacher and studio from the variant itself (`locations[0]`,
`staffMembers`), never from the title, and every class has its own capacity.
Derive the class key from the variant slug prefix, not from titles.

### Full classes: infer them from the weekly pattern

**The availability API drops a slot once it's full**, so a sold-out class
simply vanishes from `getCurrentAvailabilitiesMerged`. A studio timetable
should show it as **Full** instead. Because classes repeat weekly:

1. Fetch merged availability once per series (variant), about 6 at a time, and
   cache the result server-side for a few minutes. Exclude privates (their
   times are open studio-hours slots).
2. Turn each range into occurrences (split a range that is an exact multiple
   of the duration into back-to-back classes).
3. For each weekly series, collect its **pattern**: local weekday + local start
   time, in the studio timezone. Use local time, never UTC, so a DST change
   doesn't create two patterns for one class.
4. A pattern slot that's missing on a date between today and the **last start
   the API returned for any series** is a Full occurrence (`inferred: true`).
   The cutoff keeps the end of the booking window from reading as Full. Skip
   anything inside the product's `advanceScheduleThreshold`.
5. For a Full row, show "next with space →": the next occurrence of the same
   class type, at any studio, that isn't full.

Never infer for workshops or privates: a full workshop just disappears.
Accepted limits: a cancelled or blocked class also reads as Full, and a series
that is full in **every** week of the horizon has no observed slot, so no
pattern, and vanishes. Keep seed density below that, or add a fallback pattern
to site content if it matters. Unit-test the inference, including the DST
weeks (`lib/timetable.test.ts` in the template).

### Booking from the timetable

Deep-link a row to the class sheet (`/timetable?occ=<seriesSlug>@<startISO>`).
The sheet shows the room plan, extras, a price preview and the policy, adds the
slot to the cart (`numberOfAttendees`, `addOns`, the UTC slot with
`staffMemberId` and `locationId`), then hands over to checkout. Customers can
move or cancel up to 12 hours before (`cancelGap` 43200 s), and a level path
in the account counts completed classes as guidance only; it never blocks a
booking.
