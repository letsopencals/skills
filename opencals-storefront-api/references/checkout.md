# Checkout: customer, addresses, consent, payment

Checkout runs on a cart (via `X-Cart-Id`) through `CheckoutService`:
`start` → `saveCustomer` → `saveAnswers` → `submit`. Read the official
templates' `app/api/checkout/*` routes for exact payloads before wiring payment.

## Save the customer (`saveCustomer`)

Attaches customer details to the cart and returns the resolved/created customer
id. Call it once the customer has filled the details step, **before** `submit`.

```ts
import { CheckoutService } from '@opencals/storefront-sdk';

const { data } = await CheckoutService.saveCustomer({
  headers: { 'X-Cart-Id': cartId },
  body: {
    customer: {
      kind: 'new',          // required discriminator: 'new' | 'existing'
      email: 'jo@example.com',
      firstName: 'Jo',
      lastName: 'Smith',
      phone: '+447700900000',
      billingAddress,     // optional — see below
      deliveryAddress,    // optional — see below
      marketingConsent,   // optional — see below
    },
  },
});
// data.customerId
```

`CheckoutCustomer.customer` is either a new customer (`kind: 'new'`, shape
above) or an existing one (`kind: 'existing'`, `{ kind, customerId, … }`). The
`kind` field is **required** on every customer object (`saveCustomer`, `start`,
`submit`, `AppointmentService.create`); the generated types don't show it, so add
it yourself. Without it the request fails validation. Returns `{ customerId }`.

### Existing customers when the request isn't signed in

An email or `customerId` in the body is a claim, not proof (API keys are
public). The customer is only treated as the caller when the request carries
that customer's bearer token, or when this cart's checkout created them. If a
guest enters the email of an **existing** customer without signing in:

- the booking is still attached to that customer (it shows in their account
  once they sign in, e.g. with a login code);
- nothing on their profile is overwritten (name, phone, billing, consent);
- `saveCustomer` returns `{}` with **no** `customerId`, and cart responses
  show `customer` / `customerId` as `null`. Don't rely on getting an id back:
  send the same `kind: 'new'` details again in later steps, or leave `customer`
  out (the cart already holds it);
- `submit` returns no `auth` / `customer`.

A `customerId` that isn't the signed-in customer (or the one this cart already
uses) is ignored: send an `email` with it, or the request fails with 400.

## Billing address

`billingAddress?: BillingAddress` — company name, tax registration id, and the
postal fields (`addressLine1/2`, `city`, `state`, `postalCode`, `country`, plus
optional `firstName`/`lastName`). Whether it's hidden / optional / required is a
per-store checkout setting; read the store's public settings
(`StoreService.getStorePublicSettings`) to know whether to render and require it.

Company name and tax/VAT id live on the **billing address**, not on the customer
record (Shopify-style).

Required billing fields are checked against the `billingAddress` in the request
or, if it has none, against the billing already saved on the cart. Once
`saveCustomer` has saved it, you don't need to send it again at `start` or
`submit`.

For the untrusted existing-customer case above, the billing and delivery
addresses entered are linked to that booking (order and invoice use them) with
`customerId: null` (e.g. `cart.billingAddress.customerId`), so type it as
nullable. `saveCustomer`/`start` never write them to the customer's account;
once `submit` places the order they are also added to the customer's saved
addresses.

## Delivery address

`deliveryAddress?: Address` — same postal shape. Required **only** when the cart
contains an appointment at a **DELIVERY-type** location; ignored for PHYSICAL and
ONLINE locations (those derive their address from the location itself). A
per-appointment `address` can be set at `AppointmentService.create`, but a global
delivery address supplied here at checkout **overrides** per-appointment ones.

## Marketing consent

`marketingConsent?: MarketingConsentInput` = `{ email?, sms?, whatsapp? }`,
per-channel opt-in. Set it at checkout as above, or later from the account (see
`auth-self-service.md`). Only render the channels the store has enabled.

## Checkout questions

Stores can require custom questions. Fetch them with
`CheckoutService.getCartQuestions`, collect answers, then
`CheckoutService.saveAnswers` with
`{ answers: [{ questionId, question, answer, fileIds? }] }`: `question` is the
question text (kept for the record) and is required, as is `answer` (a string).
You can also pass `checkoutQuestionAnswers` at appointment creation. Question translations: `CheckoutQuestionService.listTranslations`.

## Payment / submit

`CheckoutService.submit` finalises the order. Available payment providers and
publishable keys come from `PaymentService.getAvailableProviders` /
`getSettings` — for Stripe the backend returns the publishable key to the
storefront. When a total is £0 or below the payment minimum, the API may return a
no-payment-required provider; branch on the provider in the response rather than
assuming Stripe. The response is `{ order, auth?, customer? }`: `auth` (tokens that
sign the customer in) is only present when the request is signed in as the
order's customer or this checkout created the customer, so always treat it as
optional. Don't reimplement payment UI from scratch — reuse the template's
`components/checkout/*`.

`appointmentsSettings.markAsScheduled` on `submit` is still accepted but no
longer decides whether the booking is confirmed; the store's confirmation
policy does (below).

## Booking status after submit

The store decides whether a booking is confirmed straight away. Some stores
confirm every booking; others only bookings that meet their conditions (for
example, paid in time or booked far enough ahead) and review the rest. The
storefront API doesn't expose those conditions. Each appointment comes back
either `status: 'scheduled'` (confirmed) or `status: 'pending'` with a
`pendingReason`:

- `'awaiting_payment'`: confirmed once the order is paid (bank transfer, cash
  or invoice not yet paid, or a card payment whose capture failed).
- `'awaiting_approval'`: the merchant still has to confirm it. A paid booking
  can still be awaiting approval.

Never show a pending booking as confirmed. Label it **"Awaiting payment"** /
**"Awaiting confirmation"**, and on the thank-you page say the booking was
**received**, not confirmed. Branch on `pendingReason` only.

Deadlines live on the order. After `paymentDueAt` an unpaid booking is either
cancelled or handed to the merchant to review (it becomes
`'awaiting_approval'`); cash orders never get one. After `approvalDueAt` a
booking still awaiting approval is cancelled. Both are null when no deadline
applies. A cancelled order has `cancelledAt` and
`cancelReason` (`OrderCancelReason`: `unpaid`, `not_approved`, `declined`,
`external`, `customer`), and `paymentStatus: 'voided'` when nothing was paid
(nothing is due). Its appointments are `status: 'canceled'` with a
`cancelReason` (`unpaid`, `not_approved`, `declined`, `order_cancelled`).

A `pending` appointment **without an order** (`orderLineItem` is null) is a
cart hold, not a booking: it expires with the cart (15 minutes by default).
Don't list it in the account (`status === 'pending' && !orderLineItem` → skip).

Orders no longer carry `logs` (merchant-only since SDK 0.3.20); drop any read
of `order.logs`.

## Manual payment: `paymentInstructions`

Manual providers (`bank_transfer`, `cash`, `stripe_invoice` — "Pay by invoice")
create the order without taking money. When something is still due, `submit`
also returns `paymentInstructions` (also on an idempotent re-submit):

```ts
const { data } = await CheckoutService.submit({ headers: { 'X-Cart-Id': cartId }, body });
const pi = data.paymentInstructions; // { provider, hostedInvoiceUrl?, dueAt?, bankDetails? } | undefined
```

Render it on the thank-you page:

- `stripe_invoice`: a **"Pay invoice"** link to `hostedInvoiceUrl`, only if it
  starts with `https://` (it can be null while the invoice is being issued).
- `bank_transfer`: `bankDetails` (`accountName`, `bankName`, `iban`, `swift`;
  each nullable).
- `cash`: a "pay in person" note.
- `dueAt`, when set: "Pay by …" in the customer's timezone. After it, the
  unpaid order is cancelled.

It's absent for card payments and paid orders.

## Checkout conflicts (409)

`start` and `submit` reject a cart that can't be checked out with **409** and an
error `code` (`OpencalsApiError.code`):

- `CART_EXPIRED`: a held time is gone. `start` and `submit` first revive an
  expired cart (they re-hold each slot that is still free and carry on, so a
  customer who paid just after the countdown still gets the booking); only when
  a slot was taken meanwhile is it a 409, and an uncaptured card payment is
  released. Start a new cart and ask the customer to re-pick (see
  `cart-lifecycle.md`).
- `CART_ALREADY_CHECKED_OUT` (`start` only): the cart is already an order; the
  error body carries its `orderId`. Send the customer to that order's
  confirmation instead of checking out again. A repeated `submit` for such a
  cart returns that order.
- `CHECKOUT_IN_PROGRESS`: another `start` or `submit` of the same cart (a
  double click, a second tab) is still running. Keep the cart, tell the
  customer it's being processed, and retry `submit` after a moment; once the
  order exists it returns that order.

## Unauthorised card payment (402)

`submit` answers **402** with `code: 'PAYMENT_NOT_AUTHORIZED'` and the
provider's `paymentStatus` (e.g. `requires_payment_method`, `requires_action`)
when the card payment was never authorised: declined, 3-D Secure not completed,
or cancelled. No order is placed and the cart is untouched. Show the card error,
let the customer confirm the payment again (or with another card), then call
`submit` again with the same cart.
