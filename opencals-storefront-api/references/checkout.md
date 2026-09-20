# Checkout: customer, addresses, consent, payment

Checkout runs on a cart (via `X-Cart-Id`) through `CheckoutService`:
`start` → `saveCustomer` → `saveAnswers` → `submit`. Read the shipping
templates' `app/api/checkout/*` (or storefront `actions/checkout/*`) routes for
exact payloads before wiring payment.

## Save the customer (`saveCustomer`)

Attaches customer details to the cart and returns the resolved/created customer
id. Call it once the customer has filled the details step, **before** `submit`.

```ts
import { CheckoutService } from '@opencals/storefront-sdk';

const { data } = await CheckoutService.saveCustomer({
  headers: { 'X-Cart-Id': cartId },
  body: {
    customer: {
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

`CheckoutCustomer.customer` is either a new customer (shape above) or an existing
one (`ExistingOrderCustomer`, referencing a customer id). Returns
`{ customerId }`.

## Billing address

`billingAddress?: BillingAddress` — company name, tax registration id, and the
postal fields (`addressLine1/2`, `city`, `state`, `postalCode`, `country`, plus
optional `firstName`/`lastName`). Whether it's hidden / optional / required is a
per-store checkout setting; read the store's public settings
(`StoreService.getStorePublicSettings`) to know whether to render and require it.

Company name and tax/VAT id live on the **billing address**, not on the customer
record (Shopify-style).

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
`CheckoutService.saveAnswers` (or pass `checkoutQuestionAnswers` at appointment
creation). Question translations: `CheckoutQuestionService.listTranslations`.

## Payment / submit

`CheckoutService.submit` finalises the order. Available payment providers and
publishable keys come from `PaymentService.getAvailableProviders` /
`getSettings` — for Stripe the backend returns the publishable key to the
storefront. When a total is £0 or below the payment minimum, the API may return a
no-payment-required provider; branch on the provider in the response rather than
assuming Stripe. Don't reimplement payment UI from scratch — reuse the template's
`components/checkout/*`.
