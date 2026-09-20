# Setup, services & error handling

## Initialise once

`setupOpencals` configures a single global client. Call it once (a side-effect
import is the cleanest pattern) and import service classes anywhere afterwards.

```ts
// lib/opencals.ts
import { setupOpencals } from '@opencals/storefront-sdk';

setupOpencals({
  baseUrl: process.env.OPENCALS_API_URL ?? 'https://api.opencals.com',
  apiKey: process.env.OPENCALS_API_KEY, // storefront key: sfk_…
  logging: process.env.NODE_ENV === 'development',
  // throwOnError: true, // see the gotcha below
});
```

```ts
// any server file that calls the API:
import '@/lib/opencals';
import { ProductService } from '@opencals/storefront-sdk';
```

The API key is a **storefront** key (`sfk_…`) generated in the Opencals
dashboard. Keep it server-side; it authorises read + booking operations for one
store. Customer-account calls additionally need the customer's bearer token (see
`auth-self-service.md`).

## Service classes (import the class, call the static method)

The SDK is class-based; every operation is a static method whose name is the
operationId. Method arguments are a single options object with `path`, `query`,
`body` and `headers` as needed.

| Class | Selected methods |
|-------|------------------|
| `ProductService` | `list`, `get`, `getBySlug`, `getByExternalId`, `getByExternalVariantId`, `getCurrentAvailabilities`, `getCurrentAvailabilitiesMerged`, `getNearestAvailability`, `listAddOns`, `listAddOnsBySlug` |
| `CartService` | `createOrGet`, `get`, `addItem`, `removeItem`, `addAddOn`, `removeAddOn`, `updateAddOnQuantity`, `applyCode`, `removeCode`, `extendExpiration` |
| `AppointmentService` | `create`, `list`, `find`, `findByExternalOrderName`, `reschedule`, `cancel`, `feedback`, `addGuest`, `removeGuest`, `getBookingPreferences` |
| `CheckoutService` | `start`, `saveCustomer`, `saveAnswers`, `getCartQuestions`, `submit` |
| `AuthService` | `signIn`, `signUp`, `oauth`, `refresh`, `requestLoginCode`, `verifyLoginCode`, `requestEmailVerification`, `verifyEmail`, `requestPasswordReset`, `resetPassword`, `resolveLink` |
| `SelfService` | `getProfile`, `updateProfile`, `changePassword`, `getMarketingConsent`, `updateMarketingConsent` |
| `OrderService` | `list`, `find` |
| `PaymentService` | `getAvailableProviders`, `getSettings` |
| `StoreService` | `getStorePublicSettings` |
| `LocationService` | `list`, `get`, `getBySlug` |
| `StaffMemberService` | `list`, `getBySlug` |
| `ProductCollectionService` | `list`, `getBySlug` |
| `AddonService` | `list`, `get`, `getBySlug` |
| `InvoiceService` | `list` |
| `FeedbackQuestionService` / `CheckoutQuestionService` | `listTranslations` |
| `CustomerUploadService` | `presign` |
| `ImageService` | `get` |

This table is a map, not the contract — confirm exact argument shapes in the
generated `.d.ts` before calling a method you haven't used here.

## API version header

The public API is versioned via the `X-Api-Version` header. The SDK sets
`X-Api-Version: 1` by default. If you call the API without the SDK (raw fetch,
another language), you **must** send this header or the versioned controllers
return an error.

## Error handling — `OpencalsApiError`

By default the SDK **returns** errors in the result object rather than throwing:

```ts
const { data, error } = await ProductService.list();
if (error) { /* handle */ }
```

If you prefer exceptions, set `throwOnError: true` in `setupOpencals`. Then
non-2xx responses throw `OpencalsApiError`, which carries the backend status
code and field-level validation details.

```ts
import { OpencalsApiError } from '@opencals/storefront-sdk';

try {
  await AppointmentService.create({ body });
} catch (err) {
  if (err instanceof OpencalsApiError) {
    // err.status, err.body — preserve them; don't collapse into a generic 500
  }
  throw err;
}
```

### `throwOnError` gotcha

If you rely on `try/catch` but forget `throwOnError: true`, the SDK will
**swallow** HTTP errors — `data` is `undefined`, no exception is thrown, and you
get confusing "undefined" downstream instead of the real 4xx. Either always
check `{ data, error }`, or set `throwOnError: true` once at setup and use
`try/catch` everywhere. Pick one convention per codebase.
