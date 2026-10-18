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
