---
name: opencals-build-booking-site
description: >-
  Build, customise and deploy a production booking website on the Opencals
  hosted API by starting from one of the official open-source Next.js templates
  (salon, barbershop, clinic, padel/squash club, luxury car rental, language
  school, villa rental, reformer Pilates studio, dental group, coworking) and the
  @opencals/storefront-sdk.
  Use when the user wants a booking/appointment/scheduling site, an Opencals
  storefront, gives you a business idea + inspiration and asks you to build the
  booking site, or says "build me a booking website". Covers picking a template,
  scaffolding, wiring services + availability + checkout, rebranding, and
  deploying to Vercel. Also covers multi-day rentals booked by the day (cars,
  equipment) via template-noir, nightly stays with fixed check-in/check-out
  (villas, cabins, rooms) via template-bali-villas, and schools with group
  classes, levels, teachers and online lessons via template-fluently, and
  fitness studios with a multi-studio class timetable, beds left and Full
  classes via template-reformer, and dental practices or clinic groups with
  many clinicians across several studios via template-alba, and coworking
  networks (desks, meeting rooms, booths, studios at several buildings) via
  template-gable. For the
  exact API/SDK call shapes, use the companion
  `opencals-storefront-api` skill.
---

# Build an Opencals booking storefront

You are helping someone stand up a real, bookable storefront on the Opencals
hosted booking API. You do **not** build a booking engine — availability maths,
timezones and double-booking are handled by the API. Your job is the frontend:
start from a template, point it at their store, restyle it, deploy it.

For exact SDK method names and argument shapes, use the companion
**`opencals-storefront-api`** skill — it is the ground truth for availability,
carts, appointments, checkout, discounts, self-service, guests/attendees, custom
durations and rate limiting. This skill is the build workflow around it.

## When to use this skill

Trigger when the user wants a booking, appointment, or scheduling website; an
"Opencals storefront"; hands you a business description + inspiration
(screenshots, a competitor site) and asks for the booking site. If they need a
*self-hosted* booking engine or data residency, this is the wrong tool — tell
them so and point them at Cal.com or Easy!Appointments.

## Primary journey: idea + inspiration → live site

1. **Understand the business.** What do they sell time for — people's time
   (stylist, clinician, coach) or *things* (courts, rooms, lanes, equipment), or
   both? Group sessions? Multiple locations? Online/delivery? This decides the
   template and the modelling.
2. **Pick the closest template** (below). Match the *booking shape* first, looks
   second — restyling is cheap, re-architecting isn't.
3. **Scaffold → configure → wire → rebrand → deploy** (the workflow below).
4. Pull design cues from their inspiration into the template's tokens/components;
   keep the wired booking flow intact.

If the goal is to add booking to an **existing** site rather than build a new
one, the same SDK and API apply — lean on the `opencals-storefront-api` skill and
embed the flow into their app (or an iframe/widget) instead of scaffolding a new
template.

## Prerequisites (ask for these first)

1. **What kind of business** — pick the closest starting template:
   - `template-haar` — salon (services + gallery, warm/editorial)
   - `template-frisor` — Frisør barbershop & apothecary (3 Berlin shops, three-step booking with the total always in view, retail `shelf-` add-ons sold with the booking; dataset `frisor`)
   - `template-clarity` — clinic (calm, warm; need-first Care Finder, one-page
     booking composer with per-visit intake and photo uploads, clinic or video
     visits, family care record; passwordless login)
   - `template-volt` (a.k.a. padel template) — multi-club padel & squash network
     (dark/technical): a **hybrid** of bookable resources (courts at several
     clubs, shown as a live hall map or a grid) and staff-led sessions
     (coaching), with a crew-and-kit checkout and match-ticket confirmation. Start here for any venue that rents out interchangeable units —
     courts, lanes, rooms, bays, equipment — especially alongside classes or
     lessons. See "Resource-booking businesses" below.
   - `template-noir` (NOIR Drive) — luxury car rental (black, cinematic, heavily
     animated): units booked **by the day** with a range calendar, delivery vs
     collection locations, per-day extras, a deposit at handover, plus
     chauffeur packages with staff. Start here for anything rented for one or
     more whole days — cars, boats, equipment. See "Rentals booked by the day"
     below.
   - `template-bali-villas` (Bali Villa Rentals, demo brand Lumbung Sidemen) —
     villa rental (warm, quiet, rice-field photography): villas booked **by the
     night** from check-in 15:00 to check-out 11:00, a calendar on live
     availability with turnover days, guest counts and arrival details, per-stay
     extras, change dates or cancel from the account. Start here for villas,
     cabins, guest houses or rooms. See "Nightly stays" below.
   - `template-fluently` (Fluently) — language school (colorful, playful,
     3D mascots): group classes with **level variants** and live "seats left",
     private 1:1 lessons, a free trial, teachers as staff, campuses plus an
     **online classroom** with a Join button. Start here for any school or
     course business — languages, music, coding, tutoring, art. See "Schools
     and classes" below.
   - `template-reformer` (Tensile) — reformer Pilates studio in Berlin
     (technical, warm; concrete, ink and spring colours): a weekly timetable
     across three studios plus online where every class names its instructor
     and shows **beds left**, Full classes with "next with space", a class
     sheet that hands over to checkout, 1:1 and duet privates, dated workshops,
     waiver questions, intro and off-peak discounts, move/cancel and Join from
     the account. Start here for Pilates, yoga, barre, spin or any studio with
     a class timetable. See "Fitness and Pilates studios" below.
   - `template-alba` (Alba Dental) — dental group in London (light, precise;
     porcelain, lagoon teal, dental-chart annotations): five studios plus video
     consultations, 26 clinicians booked by name or **first available**, a
     Smile Check questionnaire on a tooth chart, a chair-time booking planner
     where the hygienist straight after is a second cart appointment,
     per-patient pre-visit questions, a recall reminder, and Alba Kids, a
     separate children's site. Start here for dental, physio or any clinic
     group with many clinicians at several locations.
   - `template-gable` (Gable) — coworking network in Dublin, Cork and Galway
     (drawing-set look; cobalt, marigold, hatched busy time): six houses with
     flex desks booked **by the seat** (one capacity product per house), studio
     desks, meeting rooms, phone booths, studios, event spaces and day offices,
     all staffless units with custom duration; a search-style planner with one
     cached availability read per house, a **day sheet** cart that puts a desk,
     a room and a booth on one checkout and suggests extras in the gaps,
     meeting invitees as guests, and a Week plan account. Start here for
     coworking, meeting-room hire, studios or any venue renting rooms by the
     half hour at several sites. See "Resource-booking businesses" below.
2. **An Opencals private storefront key** (`spk_...`) — from the Opencals
   dashboard (Users → your user → API Keys). The templates call the API from the
   server, so the key stays secret and each visitor gets their own rate limits.
   If they don't have one, direct them to create a store and a private storefront
   key before deploying (the site builds without it, but won't return live data).
3. Node.js 20+ and a Vercel account (free tier is fine) for deployment.

## Workflow

### 1. Scaffold from the closest template

```bash
npx create-next-app -e https://github.com/letsopencals/template-haar my-booking-site
cd my-booking-site
npm install
```

Swap the repo for `template-frisor`, `template-clarity`, `template-volt`,
`template-noir`, `template-bali-villas`, `template-fluently`, `template-reformer`, `template-alba` or `template-gable` as appropriate. These are complete apps — services catalogue, real-time
availability, cart, checkout, Stripe payments and customer accounts are already
wired. Do not rebuild them.

### 2. Configure environment

Copy `.env.example` to `.env.local` and fill in:

```
OPENCALS_API_KEY=spk_your_key_here          # Required — the private storefront key
AUTH_SECRET=<random string>                 # Required — session encryption
# OPENCALS_API_URL=https://api.opencals.com # Optional — defaults to this
```

Generate `AUTH_SECRET` with `openssl rand -base64 32`. There is no Stripe env var: checkout start returns the `publishableKey` (matching the store's test/live mode) with the `clientSecret` and `stripeAccountId`.

### 3. Understand the SDK wiring (do not reinvent it)

The SDK is initialised once via a side-effect import in `lib/opencals.ts`, which
every API route imports at the top. Services are class-based
(`ProductService`, `CartService`, `AppointmentService`, `CheckoutService`,
`AuthService`, `SelfService`, …). See `references/booking-flow.md` for the
build-level overview, and the `opencals-storefront-api` skill for the exact
call shapes — **use those, do not invent method names.**

### 4. Rebrand and customise

Mostly design, not plumbing: colour/font tokens in `app/globals.css`, copy and
business details in `lib/site-config.ts`; services/staff/locations come from the
store via the API. See `references/customizing.md`.

### 5. Verify locally

```bash
npm run dev
```

Confirm services load, availability returns slots, and a test booking completes.
API routes surface real errors via `handleApiError` — read them, don't swallow.

### 6. Deploy to Vercel

```bash
npx vercel
```

Add the same env vars in the Vercel project settings, then `npx vercel --prod`.
Free tier is fine for launch; the user pays Opencals only for bookings.

## Resource-booking businesses (courts, lanes, rooms, equipment)

Some venues book *things*, often many interchangeable units, sometimes alongside
staff-led classes. The padel/volt template is the worked example; `template-gable`
applies the same model to coworking (rooms, booths and desks at six buildings,
plus a flex floor sold by the seat through `maxAttendees`). Model each with
the conflict rule that fits — don't force everything through "staff". Custom
duration uses `allowCustomDuration`, not duration-variant products. Full details,
the three-rule decision table, multi-club modelling and the per-club availability strategy are in
`references/resource-booking.md` (and the API shapes in the
`opencals-storefront-api` skill's `custom-duration.md` / `availability.md`).

## Rentals booked by the day (cars, boats, equipment, villas)

Multi-day rentals don't use time slots. Each unit is one staffless product with
a 1-day base (`duration: 86400`, `allowCustomDuration`, `maxDuration`) on a
**continuous 24/7 schedule** (or none). The UI reads the merged availability
**ranges** (`getCurrentAvailabilitiesMerged`, no `duration`), checks on the
client that pick-up → return fits inside one range, and books a slot from
pick-up 00:00 to return 00:00 in the store timezone, converted to UTC. Handover
windows and flight numbers travel as appointment `customAttributes`; the
deposit is a required checkbox, held at handover. `template-noir` is the worked
example; full pattern in `references/rentals-multi-day.md`.

## Nightly stays (villas, cabins, guest houses, rooms)

Same model as day rentals, plus `fixedTimes` with `fixedStartTime` (check-in,
e.g. 15:00) and `fixedEndTime` (check-out, e.g. 11:00). The slot runs from the
check-in date at `fixedStartTime` to the check-out date at `fixedEndTime` in the
schedule timezone, converted to UTC; build it server-side from the product's own
fixed times. The API **enforces** this for customer bookings and reschedules
(`400` otherwise; merchant bookings are exempt) and prices `ceil(booked / base)`,
so N nights = N × price. A check-out day can be the next check-in day. Minimum
nights live in site content, and guest counts, arrival time and flight number go
in `customAttributes`. `template-bali-villas` is the worked example; full
pattern in `references/rentals-multi-day.md` §11.

## Schools and classes (languages, music, coding, tutoring)

No special engine features: **subjects are collections**, **a class is a
product whose variants are levels** (group, `maxAttendees > 1`) or lengths
(private, 1 seat), a $0 1-seat product is a free trial, and **teachers are
staff** assigned per class. The format (online vs campus group) is derived from
the class's location types, so nothing is keyed off slugs. Live "seats left"
comes from slot availability. The **Online location's `link`** is the classroom
URL, returned on the booked appointment's location, which powers a "Join lesson"
button. `template-fluently` is the worked example; full pattern in
`references/classes-and-lessons.md`.

## Fitness and Pilates studios (reformer, yoga, barre, spin)

Same class model, one level deeper: **one variant per class × studio ×
instructor** (`flow-mitte-lena`), each with its own weekly schedule, its studio
as the only location, its instructor as the only staff member, and
`maxAttendees` = the beds in the room. Rooms and bed plans are template
content. **The availability API drops a slot once it's full**, so a timetable
learns each series' weekly pattern and shows a missing week as **Full**, with a
"next with space" link; dated workshops just disappear when full. Privates
(1:1 and duet) are one seat each and share a product pool per studio; a duet
is one booking with a `people: "2"` custom attribute. `template-reformer` is
the worked example; full pattern in `references/classes-and-lessons.md` §6.

## Resources

- [Opencals](https://opencals.com) — main site
- [Docs & API reference](https://opencals.com/docs)
- [@opencals/storefront-sdk on npm](https://www.npmjs.com/package/@opencals/storefront-sdk)
- [Dashboard](https://app.opencals.com) — create stores, manage storefront API keys

## Guardrails

- Never hard-code services, prices or availability in the frontend — they come
  from the API.
- Never invent SDK method names. Use the `opencals-storefront-api` skill or the
  template's existing `app/api/` routes.
- The frontend is MIT-licensed: it's fine to strip all Opencals branding from
  the UI. The API itself is not open source and is not part of what you ship.
