# Rate limiting

The API throttles per storefront key + client IP. Over the limit → **HTTP 429**,
surfaced as `OpencalsApiError` with `status === 429`. Bursts are the usual cause:
fanning out one availability call per unit for a grid, or a calendar hydrating
many products at once.

Note: the API does not reliably surface a `Retry-After` header to the SDK — use
your own backoff rather than depending on it.

## Canonical client pattern: retry with exponential backoff + jitter

This is the shipped pattern from
`widgets/podcast-house-booking-widget/src/api/rateLimit.ts`. Wrap every
fan-out / burst-prone SDK call in it.

```ts
const MAX_ATTEMPTS = 4;
const BASE_DELAY_MS = 500;
const MAX_DELAY_MS = 8000;

const delay = (ms: number) => new Promise((r) => setTimeout(r, ms));

function isRateLimited(err: unknown): boolean {
  return !!err && typeof err === 'object' && 'status' in err && (err as any).status === 429;
}

export async function withRateLimitRetry<T>(
  operation: () => Promise<T>,
  label = 'request',
): Promise<T> {
  for (let attempt = 1; ; attempt += 1) {
    try {
      return await operation();
    } catch (error) {
      if (!isRateLimited(error) || attempt >= MAX_ATTEMPTS) throw error; // final 429 propagates
      const backoff = Math.min(MAX_DELAY_MS, BASE_DELAY_MS * 2 ** (attempt - 1)); // 500,1000,2000,4000
      const wait = backoff + backoff * 0.3 * Math.random(); // up to +30% jitter
      console.warn(`rate limited on ${label}; retry ${attempt}/${MAX_ATTEMPTS - 1} in ${Math.round(wait)}ms`);
      await delay(wait);
    }
  }
}
```

```ts
// usage: wrap each fanned-out availability call
const slots = await withRateLimitRetry(
  () => ProductService.getCurrentAvailabilities({ path: { productId }, query }),
  `availability ${productId}`,
);
```

## Reduce the pressure, don't just retry

- **Cache** availability per product+date(+duration); dedupe concurrent requests
  (SWR / React Query with a stable key).
- **Batch by day, not by minute** — fetch a day's slots once and derive the grid
  client-side.
- **Prefer merged ranges** (`getCurrentAvailabilitiesMerged`) for calendar
  highlighting, then fetch exact slots only for the selected day.

## API-key hardening (server side)

Storefront keys can be configured (in the dashboard) with an **allowed-origins**
allowlist (guard-enforced) and per-(key+IP) throttling. Keep the key server-side.
If you're behind a proxy/CDN, make sure the real client IP reaches the API
(trust-proxy) so throttling and origin checks key off the right address, not your
edge node.
