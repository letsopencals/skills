# Cart lifecycle & expiry

A cart holds the pending appointment(s) and add-ons before checkout. It is
identified by an id passed in the `X-Cart-Id` header, and it **expires** — held
slots are released when it does, so a stale cart can no longer be checked out.

## Create or get

```ts
import { CartService } from '@opencals/storefront-sdk';

const { data: cart } = await CartService.createOrGet({
  headers: { 'X-Cart-Id': cartId }, // omit / empty on first call → a new cart is created
});
// Persist cart.id (cookie / localStorage) and send it as X-Cart-Id on every subsequent call.
```

Cart operations all take the `X-Cart-Id` header: `addItem`, `removeItem`,
`addAddOn`, `removeAddOn`, `updateAddOnQuantity`, `applyCode`, `removeCode`,
`extendExpiration`, `get`.

## Expiry

The cart carries an `expiresAt` timestamp (the store's hold window, 15 minutes
by default; the backend owns the exact value). Two responsibilities on the client:

1. **Show a countdown.** Compute remaining time from `expiresAt` and warn the
   customer as it runs low. Recompute on tab refocus (a backgrounded tab's timer
   drifts).
2. **Extend on activity.** While the customer is actively filling checkout, call
   `extendExpiration` to push `expiresAt` out and keep the held slots.

```ts
const { data: cart } = await CartService.extendExpiration({
  headers: { 'X-Cart-Id': cartId },
});
// cart.expiresAt is now further in the future.
```

`POST /storefront/cart/extend`.

## When the cart has expired

Checkout `start` on an expired cart fails with **409** `code: 'CART_EXPIRED'`.
`submit` revives an expired cart instead (re-holds each slot that is still free)
and only fails with `CART_EXPIRED` when a slot was taken meanwhile. `start` on a
cart that already became an order fails with `code: 'CART_ALREADY_CHECKED_OUT'`
and an `orderId`; a `submit` racing another submit of the same cart gets
`CHECKOUT_IN_PROGRESS` (see `checkout.md`).

If a call fails because the cart expired, don't silently retry the same id.
Create a fresh cart (`createOrGet` with no id), re-add the items, and tell the
customer their held slot was released and to re-select — the previous slot may no
longer be free.

Appointments in a cart are `status: 'pending'` with no order: they're holds,
not bookings, and are released when the cart expires. Every store has a hold
window (15 minutes by default, set by the merchant); `extendExpiration` and
checkout `start` only ever push `expiresAt` later, never shorten it.

Reference implementation: `contexts/cart-context.tsx` in any of the official
templates (e.g. `github.com/letsopencals/template-volt`): countdown +
extend-on-activity + refocus handling.
