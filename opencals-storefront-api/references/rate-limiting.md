# Rate limiting

The API throttles per key + client IP: 50 requests/second, 300 per 10 seconds,
1,200 per minute, plus per-email and per-IP limits on sign-up, sign-in and email
endpoints (e.g. 3 login-code emails per address per 10 minutes). The client IP is
the visitor IP your server forwards with a private key (`buyerIp` /
`X-Opencals-Buyer-Ip`), else the IP the request comes from: for a server-side
site that's your server, shared by all visitors. Over the limit → **HTTP 429**,
surfaced as `OpencalsApiError` with `status === 429`. Bursts are the usual cause:
fanning out one availability call per unit for a grid, or a calendar hydrating
many products at once.

Note: the API does not reliably surface a `Retry-After` header to the SDK — use
your own backoff rather than depending on it.

## Canonical client pattern: retry with exponential backoff + jitter

Wrap every fan-out / burst-prone SDK call in it.

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

- Use a **private storefront key** (`spk_`) on the server and keep it secret.
  Never ship it to the browser (it isn't blocked there, but it's readable). It can be restricted to your
  server's IPs (allowed origins hold IPs/CIDRs for private keys); on Vercel,
  leave that empty.
- Forward the visitor's IP with `buyerIp` (Next.js: `nextBuyerIp` from
  `@opencals/storefront-sdk/next`), so throttling and abuse limits count per
  visitor, not per server. Behind another proxy/CDN, read the header it sets
  (e.g. `cloudfront-viewer-address`), never the request body.
- A **public key** (`sfk_`) belongs in browser code only, restricted to your
  domains with allowed origins; it can't forward a buyer IP.
