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
| Checkout fails after a while | Cart expired and a held slot was taken meanwhile (`start`/`submit` revive it when the slots are still free) | Countdown + `extendExpiration` on activity; recreate cart + re-select on `CART_EXPIRED` |
| 409 `CART_EXPIRED` / `CART_ALREADY_CHECKED_OUT` at checkout | A held slot was taken after the cart expired, or the cart already became an order (body has `orderId`) | Start a new cart; or show the existing order — see `checkout.md` |
| 409 `CHECKOUT_IN_PROGRESS` on start/submit | Another checkout of the same cart (double click, second tab) is still running | Keep the cart, show "processing", retry `submit` shortly — it returns the order once placed |
| 402 `PAYMENT_NOT_AUTHORIZED` on submit | The card payment isn't authorised (`paymentStatus` e.g. `requires_payment_method` after a decline, `requires_action` when 3-D Secure wasn't completed) | Cart is untouched: have the customer confirm the payment again (or another card), then `submit` again |
| 400 `PAYMENT_PROVIDER_UNAVAILABLE` on start | Hard-coded a provider (e.g. `cash`) the store hasn't enabled | Offer only `getAvailableProviders`; route on `checkoutMode` — see `checkout.md` |
| 400 `PAYMENT_REFERENCE_MISMATCH` on submit | Sent a PaymentIntent from another cart or from before the cart total changed | Call `start` again and confirm the new `clientSecret` |
| Unpaid / unapproved bookings shown as "Confirmed" | Ignoring `status: 'pending'` + `pendingReason` | Label "Awaiting payment" / "Awaiting confirmation"; thank-you says "received" — see `checkout.md` |
| Account lists bookings the customer never placed | Cart holds (pending, no order) listed | Skip `status === 'pending' && !orderLineItem` |
| Slots show at the wrong hour | Treating UTC slot times as local | All slot times are UTC; pass `timezone` on availability, convert for display only |
| Customer can't reschedule into their own slot | Availability not excluding the moved appointment | Pass `excludeAppointmentId` — see `reschedule-cancel.md` |
| Bursts of 429 on a grid/calendar | Un-throttled fan-out | Wrap in `withRateLimitRetry` + cache — see `rate-limiting.md` |
| A service/product isn't showing | Dashboard config (inactive product, no availability, wrong location) | Verify in the dashboard, not in code |
| 401 on account calls | Missing customer bearer token | Storefront reads use the API key; account calls also need `Authorization` — see `auth-self-service.md` |
| 401 `This API key can't be used here` | An admin key (`sk_`) was sent to the storefront API | Use a storefront key: `spk_` on the server, `sfk_` in the browser |
| 401 `Request IP not allowed for this API key` | The private key is restricted to server IPs | Add your server's outgoing IP to the key, or clear the list on hosts without fixed IPs |
| Reschedule/cancel rejected | Inside `rescheduleGap` / `cancelGap` | Read the gap and disable the action when too close |
| Discount code "works" but total unchanged | Re-deriving totals from list price | Read the cart's discounted totals; they're authoritative — see `discounts.md` |

## Where to look

- Exact request/response shapes: the generated SDK types
  (`@opencals/storefront-sdk` `.d.ts`, or source `src/client/{sdk,types,zod}.gen.ts`
  in `github.com/letsopencals/storefront-sdk`).
- Working call sequences: the official templates' `app/api/*` routes and
  `contexts/cart-context.tsx`.
- Enable `logging: true` in `setupOpencals` during development to see requests.
