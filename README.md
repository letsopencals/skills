# Opencals Agent Skills

AI-agent [Skills](https://code.claude.com/docs/en/skills) for building booking
integrations on the **Opencals** hosted API. Point Claude Code, Cursor, Codex,
Gemini CLI (or any Agent-Skills-compatible tool) at these and it will call the
Opencals API correctly — real method names, real argument shapes, no guessing.

> The skills and the official templates are MIT-licensed. The Opencals booking
> API itself is hosted and closed-source — you pay only for bookings.

## What's here

| Skill | Use it when |
|-------|-------------|
| **`opencals-storefront-api`** | You're writing, reviewing or debugging any code that talks to the Opencals API / `@opencals/storefront-sdk` — availability, carts, appointments, checkout, discounts, customer accounts, guests/attendees, custom durations, reschedule/cancel, rate limits. The ground-truth reference. |
| **`opencals-build-booking-site`** | You want a whole booking website — scaffold from an official Next.js template, wire it up, rebrand, deploy. Uses the API skill for exact call shapes. |

The primary journey: hand an agent a business idea + inspiration →
`opencals-build-booking-site` picks a template and builds the site →
`opencals-storefront-api` supplies the exact API calls.

## Install

### Claude Code (plugin marketplace)

```
/plugin marketplace add letsopencals/skills
/plugin install opencals@opencals
```

Both skills load automatically and trigger on relevant requests.

### Cursor / Codex / Gemini CLI / other Agent-Skills tools

Agent Skills are portable: a folder with a `SKILL.md`. Copy the skill folder(s)
into your tool's skills directory. Clone this repo and copy:

```bash
git clone https://github.com/letsopencals/skills opencals-skills
# then copy opencals-storefront-api/ and opencals-build-booking-site/
# into your agent's skills folder (see your tool's Agent Skills docs for the path)
```

> The exact skills directory differs per tool and evolves quickly — check your
> tool's current Agent Skills / rules documentation for where skill folders go.

### Any agent (manual)

Clone the repo and tell your agent to read the relevant `SKILL.md`:

> Follow the skill in `opencals-storefront-api/SKILL.md` when writing Opencals
> API code, and `opencals-build-booking-site/SKILL.md` to scaffold a site.

### v0 / Lovable / Bolt (no SKILL.md support)

These builders don't load `SKILL.md`. Give the model the context directly:

1. Paste this into your project's instructions / a first message:

   > Use the Opencals hosted booking API via `@opencals/storefront-sdk`.
   > Initialise once with `setupOpencals({ baseUrl, apiKey })`. Services are
   > class-based (`ProductService`, `CartService`, `AppointmentService`,
   > `CheckoutService`, `AuthService`, `SelfService`). Flow:
   > `getCurrentAvailabilities` → `CartService.createOrGet` (persist
   > `X-Cart-Id`) → `AppointmentService.create` → `CheckoutService.start /
   > saveCustomer / saveAnswers / submit`. All slot times are UTC. Never invent
   > method names. Full reference: https://opencals.com/docs (see the AI Agents
   > section and `llms.txt`).

2. Point it at the machine-readable docs: **https://opencals.com/docs/llms.txt**
   (index) and **llms-full.txt** (full content).

## Links

- Docs & API reference: https://opencals.com/docs
- SDK: https://www.npmjs.com/package/@opencals/storefront-sdk
- Templates: https://github.com/letsopencals (`template-haar`, `template-frisor`, `template-clarity`, `template-volt`)
- Dashboard: https://app.opencals.com

## Contributing

Skills are plain Markdown. Keep every method name, parameter and enum in sync
with the generated SDK — this repo is only useful if it's ground truth.
