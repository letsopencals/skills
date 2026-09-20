# Customer accounts: auth & self-service

Two layers:

- **`AuthService`** — sign a customer in / up and obtain tokens.
- **`SelfService`** — act as the signed-in customer (their profile, consent).
  Also, `AppointmentService` / `OrderService` list-and-find return the caller's
  own records when called with the customer's bearer token.

Storefront **read** and **booking** calls use the `sfk_…` API key.
**Customer-account** calls additionally require the customer's bearer token
(from sign-in) in the `Authorization` header. In the templates this is wired via
a `requireAuth()` helper returning `{ headers: { Authorization } }`.

## Sign in / sign up

```ts
import { AuthService } from '@opencals/storefront-sdk';

const { data } = await AuthService.signIn({ body: { email, password } });
// tokens come back in data; persist per your session strategy, then refresh()
```

`AuthService` methods: `signIn`, `signUp`, `oauth` (OAuth provider),
`refresh` (renew tokens), `requestPasswordReset` / `resetPassword`,
`requestEmailVerification` / `verifyEmail`, `resolveLink` (resolve an
emailed link/token), and the passwordless pair below. Confirm each body shape in
the SDK's generated types.

## Passwordless (6-digit code)

Primary login for some templates (e.g. the clinic): the customer enters their
email, receives a code, and verifies it.

```ts
await AuthService.requestLoginCode({ body: { email } });
// … customer enters the 6-digit code …
const { data } = await AuthService.verifyLoginCode({ body: { email, code } });
```

There is also a magic-link path (`resolveLink`) for emailed links — templates
expose it at a `/link/[token]` route.

## Sessions / refresh

Access tokens are short-lived; use `AuthService.refresh` to renew. Store tokens
server-side (e.g. an encrypted session) rather than in client-accessible storage.
The templates use `AUTH_SECRET` to encrypt the session.

## Self-service (the signed-in customer)

```ts
import { SelfService } from '@opencals/storefront-sdk';

const { data: profile } = await SelfService.getProfile({ headers: authHeaders });
await SelfService.updateProfile({ headers: authHeaders, body: { /* … */ } });
await SelfService.changePassword({ headers: authHeaders, body: { /* … */ } });

// Marketing consent (per-channel)
const { data: consent } = await SelfService.getMarketingConsent({ headers: authHeaders });
await SelfService.updateMarketingConsent({
  headers: authHeaders,
  body: { email: true, sms: false, whatsapp: true },
});
```

The customer's appointments and orders are read via `AppointmentService.list` /
`find` and `OrderService.list` / `find` with the customer's bearer token — that
scopes results to them. Rescheduling and cancellation of their own appointments:
see `reschedule-cancel.md`.

> `saveCustomer` (checkout) vs sign-up: `saveCustomer` attaches customer details
> to a **cart** for an order and may create a lightweight customer without a
> password — call it during checkout. `AuthService.signUp` creates a
> **login-capable account**. A guest who checked out can later be turned into a
> full account.
