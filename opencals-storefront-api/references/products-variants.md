# Products, variants and images

## The model: read the variant, not the product

Every bookable thing is a **variant**, and a variant is a full product (own
`id`, `slug`, `price`, `duration`, staff, locations, images). The API groups
variants by `productGroupId` and returns the group as one item whose
`variants` array holds them all. A product with no siblings still comes back
this way: its `variants` holds exactly one entry, itself (`variants[0].id ===
product.id`).

So the top-level item is a grouping shell. **Take the data you render or book
from the variant**: the one the customer picked, or for a single-variant
product the variant whose `id` equals the product's `id`.

```ts
import { ProductService, type ProductListItemResponse } from '@opencals/storefront-sdk';

const { data } = await ProductService.list({ query: { take: 50 }, throwOnError: true });
const products = data.data; // paginated: { data: ProductListItemResponse[], ... }

function ownVariant(p: ProductListItemResponse) {
  return p.variants?.find((v) => v.id === p.id) ?? p.variants?.[0];
}
```

Book with the variant's `id` as `productId`.

## Images

What each endpoint returns:

| Endpoint | Top-level item | Each variant |
|----------|----------------|--------------|
| `ProductService.list` (`GET /storefront/products`) | `imageId` only, **no `image`** | `imageId`, `image`, `images` |
| `ProductService.getBySlug` (`GET /storefront/products/slug/{slug}`) | `imageId`, `image` | `imageId`, `image`, `images` |
| `ProductService.get` (`GET /storefront/products/{productId}`) | `imageId`, `image`, `images` | `imageId`, `image` |

- `image` is the **default image** picked in the dashboard, and `imageId` is
  its id. `images` is the product's full image set, and the default is part of it.
- **`images` has no defined order.** The API returns it in whatever order the
  database does, so `images[0]` is *not* the default and can change between
  calls. Never treat it as the main photo. Resolve the default from `image`, or
  failing that `images.find((i) => i.id === imageId)`.
- Build a gallery as the default first, then the rest, deduped. Sort the rest
  (for example by URL) if you need a stable order.

```ts
type Img = { id?: string; url?: string | null };

function gallery(v: { imageId?: string | null; image?: Img | null; images?: Img[] | null }): string[] {
  const images = v.images ?? [];
  const main = v.image?.url ?? images.find((i) => i.id === v.imageId)?.url;
  const rest = images.map((i) => i.url).filter((u): u is string => Boolean(u)).sort();
  return [...new Set([main, ...rest].filter((u): u is string => Boolean(u)))];
}

const photos = gallery(ownVariant(product) ?? {}); // photos[0] is the main image
```

Product photos, the store logo and the banner (`StoreService.getStorePublicSettings`
→ `storefrontSettings.logoImage` / `bannerImage`) are managed in the dashboard.
Render them from the API, and don't ship copies in the site's `public/` folder.
