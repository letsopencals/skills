# Discounts

Three kinds, all resolved server-side and reflected in the cart/order totals.
Never compute discounts yourself — read them off the cart.

## 1. Automatic discounts

Configured in the dashboard with conditions (e.g. "10% off bookings over 2
hours"). Applied automatically when the cart matches — **no client call**. They
appear in the cart response as applied discounts. Just render them.

## 2. Manual discount codes (promo codes)

The customer enters a code; you apply it to the cart.

```ts
import { CartService } from '@opencals/storefront-sdk';

await CartService.applyCode({
  headers: { 'X-Cart-Id': cartId },
  body: { code: 'SUMMER10' },
});

await CartService.removeCode({ headers: { 'X-Cart-Id': cartId } });
```

`applyCode` can return **400** with a machine-readable reason — map it to a
friendly message and keep the customer in the flow:

```ts
type DiscountCodeErrorResponse = {
  message: string;      // human-readable
  code:
    | 'CODE_NOT_FOUND'
    | 'CODE_EXPIRED'
    | 'CODE_DISABLED'
    | 'CODE_LIMIT_REACHED'
    | 'CODE_CUSTOMER_INELIGIBLE'
    | 'CODE_NO_MATCHING_ITEMS'
    | 'CODE_NO_CREDIT'
    | 'minimum_requirement_not_met'
    | 'MINIMUM_DURATION_NOT_MET'   // SDK 0.3.21+
    | 'MAXIMUM_DURATION_EXCEEDED'; // SDK 0.3.21+
  statusCode: number;
};
```

A code's booked-time limits count the **whole order** (the total booked time of the bookings it covers, not per
appointment and not multiplied by quantity or attendees). Too little → `MINIMUM_DURATION_NOT_MET`, too much →
`MAXIMUM_DURATION_EXCEEDED`; the `message` already says what's needed ("Book at least 2 hours…"). If you key a
`Record` by the code type, add both keys when you move to 0.3.21, or fall back to `message` for unknown codes.

## 3. Balance-based (credit) discounts

A customer can hold a **balance** granted in the dashboard, spent automatically
against eligible items. The balance is either:

- **Monetary** — a money credit, or
- **Time / duration** — e.g. "10 free hours", surfaced with
  `valueType: 'time'`.

Balance-based discounts show up **per line item** in the cart:

```ts
type CartItemAppliedDiscount = {
  discountId: string;
  title: string;
  amountPerUnit: number;
  valueType: 'percentage' | 'fixed_amount' | 'time'; // 'time' = duration/credit balance
  value: number;
  targetType: 'base' | 'add_on'; // whether it hits the base service or an add-on
};
```

`CODE_NO_CREDIT` is the error when a credit-backed code has no remaining balance.

## Stacking

Discounts are either exclusive or **combinable**. Per line item, only the single
largest exclusive discount applies; combinable discounts apply on top. A
discount that only targets add-ons (`targetType: 'add_on'`) is usually set up as
combinable so it stacks with the line's best offer, and it can come back with a
zero amount on a line that has none of those add-ons. Hide zero-amount
discounts rather than showing "-$0.00".

## Where discounts appear in the cart

- **Per item**: `item.discounts: CartItemAppliedDiscount[]` (includes balance/time).
- **Cart-level**: applied discounts with a `title`, `totalAmount`, and `code`
  (present for code-driven ones). The cart's discounted totals are authoritative —
  display those, don't re-derive from list price.
