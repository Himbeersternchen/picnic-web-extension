@AGENTS.md

# CLAUDE.md

Additions to AGENTS.md. Where the two overlap, AGENTS.md wins.

Note: AGENTS.md links `src/lib/auth.ts`, `src/lib/picnic-client.ts` and
`src/lib/api-error.ts`; these files live under `src/lib/core/`.
It also links `src/lib/rsc/parse-rsc-page.ts`, which no longer exists: RSC pages are parsed
by picnic-api, and the app-side types are in `src/lib/rsc/rsc-page-types.ts`.

## UI conventions

Use the CSS variables and theme tokens from `src/app/globals.css` (e.g. `text-picnic-red`)
instead of hardcoded values or generic Tailwind colours.

Before writing a component, check `src/components/` for one to reuse or extend. Generic UI
primitives (modals, dialogs, overlays) belong in `src/components/ui/` so pages can share them.

## Country selection

The active country lives in the non-httpOnly `picnic_country` cookie and can be switched at
runtime; the login page and the header show one toggle per supported country. Switching
updates the cookie and reloads, so API calls, image CDN URLs and the checkout deeplink all
follow it.

- `SUPPORTED_COUNTRY_CODES` and `COUNTRY_COOKIE_NAME`: [src/lib/core/types.ts](src/lib/core/types.ts)
- Image URLs always need the country: `buildImageUrl(imageId, countryCode)` in
  [src/lib/core/image-url.ts](src/lib/core/image-url.ts)
- To add a country: add it to `SUPPORTED_COUNTRY_CODES` (and to the copy in
  `src/generate-auth-key.mjs`) and add a language file under `src/lib/core/i18n/`. The
  toggles render from the list, so no button needs touching.

## Auth

[src/proxy.ts](src/proxy.ts) gates every route except `/login`, `/api/auth/*` and Next.js
internals on the httpOnly `picnic_auth_token` cookie, and redirects to `/login?redirect=<path>`.
Login is either a raw token (`/api/auth/login`) or email/password (`/api/auth/login-credentials`,
2FA via `/api/auth/verify-2fa`).

## picnic-api quirks

- `picnic-api` is CJS (`export =`), so [src/lib/core/picnic-client.ts](src/lib/core/picnic-client.ts)
  loads it with `require()` to avoid Turbopack interop issues. Build a fresh client per
  request; don't cache instances.
- Prefer picnic-api's typed namespaces (`client.catalog`, `client.cart`, `client.app`, ...),
  e.g. `client.catalog.getSearchResults` in [src/app/api/search/route.ts](src/app/api/search/route.ts).
  For a raw page, use `getFusionPage` or `getRscPageContent` from `picnic-client.ts`; they
  throw `UnexpectedPageFormatError` when Picnic serves the other format.
- Errors are `PicnicApiError`s with the HTTP `status`, Picnic's `code` and the `body`.
  Check them with `isPicnicApiError` / `isApiAuthError`, not by matching message text.

## Fusion / PML parsing

Fusion pages are nested PML node trees. Use the traversal helpers in `src/lib/pml/` and the
shared extraction in [src/lib/product/extract-card-data.ts](src/lib/product/extract-card-data.ts)
(promotions, badges, unavailability) rather than walking nodes by hand. App types in
`src/lib/core/types.ts` are deliberately decoupled from picnic-api's types. Prices are
always in euro cents.

## Cart state

[src/contexts/cart-context.tsx](src/contexts/cart-context.tsx) updates quantities
optimistically and rolls back to the last confirmed snapshot (with a toast) on error.
Mutations for the same product are serialized by
[src/lib/cart/mutation-queue.ts](src/lib/cart/mutation-queue.ts); different products run
concurrently. `useCart()` throws outside a `CartProvider`; use `useCartOptional()` where
there may be none (e.g. the shared header).
