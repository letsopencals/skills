# Opencals Agent Skills

[Agent Skills](https://code.claude.com/docs/en/skills) that teach AI coding tools
to build booking websites and integrations on the **Opencals** API. Install them
in Claude Code, Cursor, Codex, Gemini CLI or v0, and your agent calls the
Opencals API correctly: real SDK method names, real argument shapes, no guessing.

Opencals is booking software that works like an online store. Every booking is
an order with checkout, payment, invoice and refunds attached. It handles
appointments, classes with limited seats, rentals and overnight stays. Your
agent builds the website; Opencals handles availability, double-booking
prevention, pricing, tax, discounts, Stripe payments and confirmation emails.

```bash
npx skills add letsopencals/skills
```

> The skills and the official templates are MIT-licensed. The Opencals booking
> API is hosted (not self-hostable). Development stores are free, with every
> feature and Stripe in test mode; you pay per booking once you go live.

## What's here

| Skill | Use it when |
|-------|-------------|
| **`opencals-storefront-api`** | You're writing, reviewing or debugging code that talks to the Opencals API or `@opencals/storefront-sdk`: availability, carts, appointments, checkout, discounts, customer accounts, guests and attendees, custom durations, reschedule and cancel, rate limits. The ground-truth reference. |
| **`opencals-build-booking-site`** | You want a whole booking website. It picks the closest official Next.js template, scaffolds it, wires it up, rebrands it to your inspiration and deploys it. Uses the API skill for exact call shapes. |

The usual journey: give your agent a business idea and some inspiration
(screenshots, a competitor's site, a brand feel). `opencals-build-booking-site`
picks a template and builds the site; `opencals-storefront-api` supplies the
exact API calls.

### Templates the build skill starts from

All MIT-licensed, Next.js, on GitHub under [letsopencals](https://github.com/letsopencals):

| Template | Business | Repo |
|----------|----------|------|
| Haar | Hair and beauty salon | `template-haar` |
| Frisør | Barbershop | `template-frisor` |
| Clear Care | Medical clinic | `template-clarity` |
| VOLT | Padel and squash club (courts plus coaching) | `template-volt` |
| NOIR Drive | Luxury car rental (multi-day) | `template-noir` |
| Fluently | Language school (classes and private lessons) | `template-fluently` |
| Bali Villa Rentals | Villa and holiday rentals (nightly stays) | `template-bali-villas` |
| Tensile | Reformer Pilates studio (classes with limited spots) | `template-reformer` |

## Before you start

You need an Opencals store and an API key:

1. Sign up at [app.opencals.com](https://app.opencals.com) and create a free
   **development store**. You can seed it with a sample business that matches a
   template.
2. In the dashboard, create a **private storefront key** (`spk_…`) for server
   code. A public key (`sfk_…`) with allowed origins is for browser-only embeds.

## Install

### skills.sh (Claude Code, Cursor, Codex, Gemini CLI and more)

```bash
npx skills add letsopencals/skills
```

Listed at [skills.sh/letsopencals/skills](https://skills.sh/letsopencals/skills).

### Claude Code (plugin marketplace)

```
/plugin marketplace add letsopencals/skills
/plugin install opencals@opencals
```

Both skills load automatically and trigger on relevant requests.

### Manual

Agent Skills are portable: a folder with a `SKILL.md`. Clone the repo and copy
the skill folders into your tool's skills directory:

```bash
git clone https://github.com/letsopencals/skills opencals-skills
# copy opencals-storefront-api/ and opencals-build-booking-site/
# into your agent's skills folder (see your tool's Agent Skills docs for the path)
```

Or point your agent at the files directly:

> Follow the skill in `opencals-storefront-api/SKILL.md` when writing Opencals
> API code, and `opencals-build-booking-site/SKILL.md` to scaffold a site.

## AI app builders

Each builder has a setup guide with every field ready to copy:

- **[v0](https://opencals.com/docs/ai-agents/v0)**: attach these skills from
  skills.sh (type `/` in the composer and add `letsopencals/skills`), then set
  `OPENCALS_API_KEY`.
- **[Lovable](https://opencals.com/docs/ai-agents/lovable)**: a custom connector
  plus two knowledge files.
- **[Base44](https://opencals.com/docs/ai-agents/base44)**: one backend
  function, an app secret and two workspace skills.
- **Bolt, Replit and others**: paste the
  [API reference](https://opencals.com/ai/opencals-storefront-api.md) and the
  [booking-site playbook](https://opencals.com/ai/lovable-booking-site-playbook.md)
  into the project instructions.

## Docs MCP server

The Opencals developer docs are also an MCP server, so your agent can look up
anything the skills don't cover:

```bash
claude mcp add --transport http opencals-docs https://opencals.com/docs/mcp
```

It has two read-only tools (search the docs, read a full page) and needs no
API key. It searches documentation only; store data and bookings go through
the API. Setup for Cursor and other clients:
[AI Agents overview](https://opencals.com/docs/ai-agents/overview#docs-mcp-server).

## Links

- Developer docs and API reference: https://opencals.com/docs
- Machine-readable docs: https://opencals.com/docs/llms.txt and https://opencals.com/docs/llms-full.txt
- Guide: [How to build a booking website with AI](https://opencals.com/blog/build-booking-website-with-ai)
- SDK: https://www.npmjs.com/package/@opencals/storefront-sdk
- Dashboard: https://app.opencals.com

## Contributing

Skills are plain Markdown. Keep every method name, parameter and enum in sync
with the generated SDK. This repo is only useful if it's ground truth.
