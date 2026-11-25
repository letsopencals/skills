# The booking flow (build overview)

This is the shape of the flow the templates already implement. For the **exact**
SDK method names, argument shapes and edge cases, use the companion
**`opencals-storefront-api`** skill — don't invent method names or reshape
arguments. When in doubt, read the template's own `app/api/` routes.

Every server file that calls the SDK first imports the init side-effect:

```ts
import '@/lib/opencals'; // calls setupOpencals() once
```

## The four steps

1. **Availability** — `ProductService.getCurrentAvailabilities` returns exact
   bookable slots for a date (all times UTC). `getCurrentAvailabilitiesMerged`
   returns broad windows for calendar highlighting. Pass `timezone`; pass
   `duration` (seconds, as a string) for variable-length products.
   → core skill `availability.md`.
2. **Cart** — `CartService.createOrGet`, then persist and send `X-Cart-Id` on
   every subsequent cart/checkout call. Carts expire; extend on activity.
   → core skill `cart-lifecycle.md`.
3. **Appointment** — `AppointmentService.create` from a chosen slot + `cartId`
   (+ `numberOfAttendees`, guests, add-ons as needed).
   → core skill `guests-attendees.md`.
4. **Checkout** — `CheckoutService.start → saveCustomer → saveAnswers → submit`,
   collecting customer details, billing/delivery address and marketing consent.
   → core skill `checkout.md`.

## Thank-you page and booking status

Depending on the store, a booking can come back
`status: 'pending'` with `pendingReason` `'awaiting_payment'` or
`'awaiting_approval'`. Then the thank-you page says the booking was
**received** (not confirmed) and shows "Awaiting payment" / "Awaiting
confirmation". For manual payment (bank transfer, cash, "Pay by invoice"), render
`submit`'s `paymentInstructions`: a "Pay invoice" link (https only), bank details
or a pay-in-person note, plus the due time. An expired cart is revived at
checkout when its slots are still free, so a 409 `CART_EXPIRED` means a held
slot was taken: start a new cart. A 409 `CHECKOUT_IN_PROGRESS` means it's already
being placed: keep the cart and ask the customer to wait. A 402
`PAYMENT_NOT_AUTHORIZED` on submit means the card wasn't authorised (declined,
3-D Secure not completed): keep the cart, show the card error and let them retry
payment. → core skill `checkout.md`.

## Customer-facing extras the templates ship

- **Accounts**: `AuthService` (password, passwordless code, OAuth) + `SelfService`
  (profile, marketing consent). Account pages list the customer's appointments
  and orders: skip cart holds (`pending` with no `orderLineItem`) and label
  pending bookings by `pendingReason`. → core skill `auth-self-service.md`.
- **Reschedule / cancel**: `AppointmentService.reschedule` / `cancel`, gated by
  `rescheduleGap` / `cancelGap`; reschedule availability must pass
  `excludeAppointmentId`. → core skill `reschedule-cancel.md`.
- **Discounts**: automatic, promo codes (`CartService.applyCode`), and
  balance/credit (`valueType: 'time'`). → core skill `discounts.md`.

## Error handling

The SDK surfaces `OpencalsApiError` with the backend status and field-level
validation. Templates route every catch through a shared `handleApiError(err)`
(`lib/api-error-handler.ts`) that preserves them — reuse it, never return a
generic "Internal server error". Handle 429s with backoff (core skill
`rate-limiting.md`).
