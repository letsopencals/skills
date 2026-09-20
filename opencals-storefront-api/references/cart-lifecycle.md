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

The cart carries an `expiresAt` timestamp (roughly a few minutes out; the backend
owns the exact window). Two responsibilities on the client:

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

If a call fails because the cart expired, don't silently retry the same id.
Create a fresh cart (`createOrGet` with no id), re-add the items, and tell the
customer their held slot was released and to re-select — the previous slot may no
longer be free.

Reference implementations: `frontend/apps/storefront/contexts/cart-context.tsx`
and `templates/padel-club-template/contexts/cart-context.tsx` (countdown +
extend-on-activity + refocus handling).
