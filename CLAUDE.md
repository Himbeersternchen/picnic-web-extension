# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Unofficial web interface for the Picnic online supermarket, built with Next.js 16 (App Router), React 19, TypeScript 5, and Tailwind CSS 4. Wraps the [`picnic-api`](https://github.com/MRVDH/picnic-api) npm package in a server-side proxy layer so the Picnic API token never reaches the browser.

## Commands

```bash
npm run dev      # development server
npm run build    # production build
npm run start    # start production server
npm run lint     # ESLint (no test suite)
```

There is no test suite — `npm run lint` is the only automated check.

## Architecture

### Country selection

The active Picnic country is stored in a browser cookie (`picnic_country`, non-httpOnly) and can be switched at runtime — no environment variables or config files needed.

Supported values: `NL` (Netherlands), `DE` (Germany). Default: `NL`.

**Switching flow:**

- Login page shows NL / DE toggle buttons — select before logging in. The selected country is sent in the login request body and persisted as a cookie on success.
- After login, the header (`SharedHeader`) shows NL / DE toggle buttons. Clicking one updates the cookie and reloads the page so all API calls, image CDN URLs, and the checkout deeplink use the new country.

**Key files:**

- [src/lib/types.ts](src/lib/types.ts) — `SUPPORTED_COUNTRY_CODES`, `CountryCode`, `COUNTRY_COOKIE_NAME`, `getImageCdnBase()`, `parseCountryCode()`
- [src/lib/auth.ts](src/lib/auth.ts) — `readCountryCode(request)` reads the cookie server-side
- [src/lib/picnic-client.ts](src/lib/picnic-client.ts) — `buildPicnicClient(token, countryCode)`, `buildPicnicClientAnonymous(countryCode)`
- [src/lib/image-url.ts](src/lib/image-url.ts) — `buildImageUrl(imageId, countryCode)` — always requires country, no static base URL
- [src/contexts/country-context.tsx](src/contexts/country-context.tsx) — `CountryCodeProvider`, `useCountryCode()`, `useSwitchCountry()`
- [src/components/checkout-cta.tsx](src/components/checkout-cta.tsx) — `CHECKOUT_LABELS` map for localised button text

**To add a new country:** add the code to `SUPPORTED_COUNTRY_CODES` in `src/lib/types.ts` and add a label to `CHECKOUT_LABELS` in `src/components/checkout-cta.tsx`.

### Auth flow

[src/proxy.ts](src/proxy.ts) is the Next.js middleware. It gates every route (except `/login`, `/api/auth/*`, and Next.js internals) by checking for the `picnic_auth_token` HTTP-only cookie. Unauthenticated requests are redirected to `/login` with the original path preserved as `?redirect=`.

Two login methods exist under `src/app/api/auth/`:

- `POST /api/auth/login` — validates a raw Picnic auth token and sets the auth + country cookies
- `POST /api/auth/login-credentials` — authenticates with email/password, supports 2FA via `POST /api/auth/verify-2fa`

### Server-side API proxy

All `src/app/api/` route handlers run server-side. They:

1. Read the auth token from the request cookie via `readAuthToken()` ([src/lib/auth.ts](src/lib/auth.ts))
2. Read the country from the request cookie via `readCountryCode()` ([src/lib/auth.ts](src/lib/auth.ts))
3. Instantiate a fresh `PicnicClient` via `buildPicnicClient(token, countryCode)` ([src/lib/picnic-client.ts](src/lib/picnic-client.ts)) — no cached instances
4. Call the Picnic API and transform the raw response into typed domain objects
5. Return `ApiErrorResponse` with `code: "TOKEN_EXPIRED"` on auth failure (the client detects this and redirects to login)

`picnic-api` uses `export = class PicnicClient` (CJS), so it is loaded via `require()` to avoid ESM/CJS interop issues with Turbopack.

### `sendRequest` cast pattern

Several routes need the raw "Fusion page" response that `picnic-api`'s typed methods don't expose. They cast the client to access the internal `sendRequest` method directly:

```ts
const raw = await (
  client as unknown as {
    sendRequest: (
      method: string,
      path: string,
      body: unknown,
      includeFusion: boolean
    ) => Promise<unknown>;
  }
).sendRequest("GET", "/some/path", null, true);
```

The fourth argument `true` enables Picnic-specific headers that return `decorator_overrides` (bundle discounts etc.). This pattern is used in the search, cart, product, and category routes.

### PML / Fusion page parsing

The Picnic API returns "Fusion pages" — deeply nested PML (Picnic Markup Language) node trees. The `src/lib/` layer parses these into typed domain objects:

- [src/lib/pml-helpers.ts](src/lib/pml-helpers.ts) — low-level tree traversal utilities
- [src/lib/pml-product-helpers.ts](src/lib/pml-product-helpers.ts) — product-node extraction helpers
- [src/lib/parse-fusion-search.ts](src/lib/parse-fusion-search.ts) — search results → `SearchResult`
- [src/lib/parse-fusion-product.ts](src/lib/parse-fusion-product.ts) — product detail page → `ProductDetail`
- [src/lib/parse-cart.ts](src/lib/parse-cart.ts) — cart response → `CartData`
- [src/lib/parse-categories.ts](src/lib/parse-categories.ts), `parse-subcategories.ts`, `parse-category-products.ts` — category navigation
- [src/lib/extract-card-data.ts](src/lib/extract-card-data.ts) — shared extraction logic (promotions, badges, unavailability)

All application domain types are defined in [src/lib/types.ts](src/lib/types.ts), deliberately decoupled from the upstream `picnic-api` types. Prices are always in euro cents.

### Cart context and optimistic updates

[src/contexts/cart-context.tsx](src/contexts/cart-context.tsx) manages client-side cart state. Key design:

- Optimistic updates: quantity and count are updated immediately in React state before the server responds
- Per-product mutation queue ([src/lib/mutation-queue.ts](src/lib/mutation-queue.ts)): rapid +/− taps for the same product are serialized; mutations for different products run concurrently
- On server error, state rolls back to the last confirmed snapshot and a toast is shown
- `useCart()` throws if called outside a `CartProvider`; `useCartOptional()` returns `null` instead (used by `SharedHeader`)

### Pages and routing

| Route                                      | Page                                    |
| ------------------------------------------ | --------------------------------------- |
| `/`                                        | Home — shortcut grid and search bar     |
| `/login`                                   | Token or credential login               |
| `/pages`                                   | Picnic content pages (promotional)      |
| `/categories/[categoryId]`                 | Category with subcategory list          |
| `/categories/[categoryId]/[subcategoryId]` | Subcategory product grid                |
| `/product/[id]`                            | Product detail                          |
| `/cart`                                    | Shopping cart with delivery slot picker |
