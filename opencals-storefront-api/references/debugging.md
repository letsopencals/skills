# Debugging Opencals integrations

Read the error first — the API returns a real status code and, on validation
failures, field-level detail. Most "bugs" are configuration or a missing header,
not the API.

## Reading errors

- With `throwOnError: true`: catch `OpencalsApiError` and inspect `err.status`
  and `err.body`. With the default, check `{ data, error }` on every call.
- Preserve the backend status and message end-to-end. The templates route
  everything through a shared `handleApiError(err)` that keeps the status and
  validation errors instead of collapsing to a generic 500. Reuse that.

## Top recurring mistakes

| Symptom | Likely cause | Fix |
|--------|--------------|-----|
| Errors vanish, `data` is `undefined`, no throw | `throwOnError` not set but using try/catch | Set `throwOnError: true`, or always check `{ data, error }` — see `setup.md` |
| Version / "unsupported version" error on raw calls | Missing `X-Api-Version` | The SDK sends `X-Api-Version: 1`; add it manually on non-SDK calls |
| Cart calls 404 / act on the wrong cart | `X-Cart-Id` not sent or stale | Persist `cart.id`, send it on every cart/checkout call; recreate on expiry — see `cart-lifecycle.md` |
| Checkout fails after a while | Cart expired, slots released | Countdown + `extendExpiration` on activity; recreate cart + re-select on expiry |
| Slots show at the wrong hour | Treating UTC slot times as local | All slot times are UTC; pass `timezone` on availability, convert for display only |
| Customer can't reschedule into their own slot | Availability not excluding the moved appointment | Pass `excludeAppointmentId` — see `reschedule-cancel.md` |
| Bursts of 429 on a grid/calendar | Un-throttled fan-out | Wrap in `withRateLimitRetry` + cache — see `rate-limiting.md` |
| A service/product isn't showing | Dashboard config (inactive product, no availability, wrong location) | Verify in the dashboard, not in code |
| 401 on account calls | Missing customer bearer token | Storefront reads use the `sfk_` key; account calls also need `Authorization` — see `auth-self-service.md` |
| Reschedule/cancel rejected | Inside `rescheduleGap` / `cancelGap` | Read the gap and disable the action when too close |
| Discount code "works" but total unchanged | Re-deriving totals from list price | Read the cart's discounted totals; they're authoritative — see `discounts.md` |

## Where to look

- Exact request/response shapes: the generated SDK types
  (`@opencals/storefront-sdk` `.d.ts`, or source `src/client/{sdk,types,zod}.gen.ts`).
- Working call sequences: the shipping templates' `app/api/*` routes and the
  storefront app's `actions/*` and `contexts/cart-context.tsx`.
- Enable `logging: true` in `setupOpencals` during development to see requests.
