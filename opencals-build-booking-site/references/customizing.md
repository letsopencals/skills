# Customising a template

Most of the work is design and content. Booking behaviour, services, prices,
staff and availability all come from the Opencals store via the API — configure
those in the dashboard, not in code.

## Pick the closest starting point

| Template | Repo | Feel |
|----------|------|------|
| Salon | `github.com/letsopencals/template-haar` | Warm, editorial; services + gallery |
| Barbershop | `github.com/letsopencals/template-frisor` | Dark, editorial |
| Clinic | `github.com/letsopencals/template-clarity` | Department-first, medical; passwordless login |
| Padel/squash club | `github.com/letsopencals/template-volt` | Dark/technical; court grid + coach-led trainings |

Clone with `npx create-next-app -e <repo-url> my-site`.

## Branding

- **Colours** live as CSS custom properties in `app/globals.css`. Haar, for
  example, exposes `--color-accent` (`#E8530E`), `--color-accent-light`,
  `--color-accent-dark`, plus neutral tokens. Change these first — they cascade
  through the whole UI.
- **Fonts** are wired via `next/font` in the root layout.
- **Business name, tagline, contact details, social links, SEO defaults** live
  in `lib/site-config.ts` (`siteConfig`). Edit copy there rather than hunting
  through components.

## Content that comes from the API (do NOT hard-code)

- Services / products, prices, durations → dashboard.
- Staff members and their availability → dashboard.
- Locations (for multi-location stores) → dashboard.
- Store-level settings (currency, time/date format) are read at runtime via
  `StoreService` and exposed through `SettingsProvider`.

If a service isn't showing up on the site, it's almost always a dashboard
configuration issue (inactive product, no availability, wrong location), not a
code bug.

## Structural changes

The templates are ordinary Next.js App Router apps:

- Marketing/home sections are plain components under `components/` — reorder,
  add, or remove them freely.
- Pages live in `app/` (e.g. `/services`, account pages). Add routes as normal.
- Client data fetching goes through the template's API-request hook; forms
  through its form-submit hook. Follow the existing patterns instead of calling
  `fetch` directly.

## Licence

The template frontend is MIT — strip every Opencals reference from the UI,
rebrand fully, ship it for a client. The booking API itself is not open source
and is not part of what you deploy.
