# Data Model: Cart Page

**Feature**: 006-cart-page | **Date**: 2026-04-07

## Display Types (Application-Level)

These types are defined in `src/lib/types.ts` and represent the transformed cart data used by UI components. They are decoupled from the upstream Picnic API response — the raw response is `unknown` and validated/extracted at runtime by `parseCartResponse`.

### CartData

The top-level display model returned by the `/api/cart` route.

| Field               | Type              | Source (raw response path)                     | Description                                               |
| ------------------- | ----------------- | ---------------------------------------------- | --------------------------------------------------------- |
| `items`             | `CartItem[]`      | `response.items` (mapped)                      | All cart line items with decorators merged                |
| `totalPrice`        | `number`          | `response.checkout_total_price`                | Total order price in cents (includes fees and deposits)   |
| `totalCount`        | `number`          | `response.total_count`                         | Total number of items in cart                             |
| `totalDiscount`     | `number`          | Calculated                                     | Sum of per-line (`price - display_price`) in cents        |
| `depositTotal`      | `number`          | Calculated from `response.deposit_breakdown`   | Sum of `value * count` across all deposit entries         |
| `depositBreakdown`  | `DepositEntry[]`  | `response.deposit_breakdown` (mapped)          | Per-type deposit breakdown                                |
| `membershipSavings` | `number`          | `response.membership_savings`                  | Membership savings in cents                               |
| `minimumOrderValue` | `number \| null`  | Selected delivery slot's `minimum_order_value` | Minimum order value in cents, or null if unavailable      |
| `suggestions`       | `SliderProduct[]` | `response.basket_sections` (parsed)            | "Niets vergeten?" suggestions; empty array if unavailable |

### CartItem

A single line item in the cart, derived from raw order line + article objects.

| Field                    | Type              | Source (raw response path)                                       | Description                                               |
| ------------------------ | ----------------- | ---------------------------------------------------------------- | --------------------------------------------------------- |
| `id`                     | `string`          | `orderLine.id`                                                   | Line item identifier                                      |
| `productId`              | `string`          | `orderArticle.id`                                                | Product/article identifier (used for product detail link) |
| `name`                   | `string`          | `orderArticle.name`                                              | Product name                                              |
| `unitQuantity`           | `string`          | `orderArticle.unit_quantity`                                     | Unit quantity (e.g., "500 g")                             |
| `imageId`                | `string`          | `orderArticle.image_ids[0]`                                      | Primary image ID (first in array)                         |
| `displayPrice`           | `number`          | `orderLine.display_price`                                        | Current price in cents                                    |
| `originalPrice`          | `number \| null`  | `orderLine.price` if different from `display_price`, else `null` | Original price for discount display                       |
| `quantity`               | `number`          | `QUANTITY` decorator on order line                               | Quantity in cart                                          |
| `badges`                 | `Badge[]`         | Mapped from decorators                                           | Discount labels, bundle labels, freshness, base price     |
| `isUnavailable`          | `boolean`         | `UNAVAILABLE` decorator presence                                 | Whether the item is unavailable                           |
| `unavailableExplanation` | `string \| null`  | `UNAVAILABLE` decorator `.explanation.short_explanation`         | Short unavailability reason                               |
| `replacements`           | `SliderProduct[]` | `UNAVAILABLE` decorator `.replacements` (mapped)                 | Replacement product suggestions                           |

### DepositEntry

A single deposit category in the breakdown.

| Field   | Type     | Source (raw response path)  | Description                               |
| ------- | -------- | --------------------------- | ----------------------------------------- |
| `type`  | `string` | `depositBreakdown[n].type`  | Deposit category (e.g., "BAG", "DEFAULT") |
| `value` | `number` | `depositBreakdown[n].value` | Price per unit in cents                   |
| `count` | `number` | `depositBreakdown[n].count` | Number of deposit units                   |
| `total` | `number` | Calculated `value * count`  | Total deposit for this category in cents  |

### Existing Reused Types

These types already exist in `src/lib/types.ts` and are reused without modification:

- **`Badge`** — `{ text: string; variant: BadgeVariant }` — used for decorator-derived labels
- **`BadgeVariant`** — `"promo" | "discount" | "size" | "freshness" | "availability" | "info" | "unit-price"` — badge styling variants
- **`SliderProduct`** — `{ id, name, imageId, displayPrice, unitQuantity, maxCount, deposit? }` — used for replacements and "Niets vergeten?" suggestions
- **`ApiErrorResponse`** — `{ error: string; code?: AuthErrorCode }` — error response shape

## Entity Relationships

```text
CartData
├── items: CartItem[]
│   ├── badges: Badge[]               (mapped from decorators: LABEL→discount, FRESH_LABEL→freshness, BASE_PRICE→unit-price, BUNDLES_BUTTON→info)
│   └── replacements: SliderProduct[]  (mapped from UNAVAILABLE decorator replacements)
├── depositBreakdown: DepositEntry[]
└── suggestions: SliderProduct[]       (mapped from basket_sections, may be empty)
```

## Decorator → Badge Mapping

| Decorator Type   | Badge Variant  | Badge Text Source                             |
| ---------------- | -------------- | --------------------------------------------- |
| `LABEL`          | `"discount"`   | `decorator.text` (e.g., "1 + 1 gratis")       |
| `FRESH_LABEL`    | `"freshness"`  | `decorator.period` (e.g., "6 dagen vers")     |
| `BASE_PRICE`     | `"unit-price"` | `decorator.base_price_text` (e.g., "€4.81/l") |
| `BUNDLES_BUTTON` | `"info"`       | Static text: "Bundel"                         |

## Transformation Rules

### Runtime Validation Approach

Since the cart data is fetched via `sendRequest("GET", "/cart")` returning `unknown`, the `parseCartResponse` function must defensively validate the response structure at runtime. Each field access should use type guards or optional chaining with fallbacks. No picnic-api types are imported for casting.

```text
parseCartResponse(rawData: unknown) → CartData
  1. Assert rawData is an object with expected top-level fields
  2. Extract and validate each field with safe defaults
  3. Return strongly-typed CartData
```

### Price Discount Detection

```text
IF orderLine.display_price !== orderLine.price
  THEN originalPrice = orderLine.price
  ELSE originalPrice = null
```

### Total Discount Calculation

```text
totalDiscount = SUM(item.price - item.display_price) FOR EACH order line WHERE price > display_price
```

### Minimum Order Value Extraction

```text
selectedSlotId = response.selected_slot.slot_id
matchedSlot = response.delivery_slots.FIND(slot => slot.slot_id === selectedSlotId)
IF matchedSlot AND matchedSlot.minimum_order_value IS defined
  THEN minimumOrderValue = matchedSlot.minimum_order_value
  ELSE minimumOrderValue = null
```

### Decorator Override Merge

```text
FOR EACH orderLine IN response.items:
  overrides = response.decorator_overrides[orderLine.id] OR []
  FOR EACH override IN overrides:
    REPLACE decorator in orderLine.decorators WHERE decorator.type === override.type
    IF no match: APPEND override to orderLine.decorators
  REPEAT for each orderArticle in orderLine.items using orderArticle.id
```

## Validation Rules

| Rule                                                                      | Description                            | Source                   |
| ------------------------------------------------------------------------- | -------------------------------------- | ------------------------ |
| All prices are in cents (integers)                                        | No floating-point arithmetic for money | FR-007                   |
| `quantity` defaults to 1 if no `QUANTITY` decorator found                 | Graceful fallback                      | R-003                    |
| `imageId` falls back to empty string if `image_ids` is empty or missing   | Triggers placeholder image             | FR-021                   |
| `suggestions` is empty array if `basket_sections` is empty or unparseable | Graceful degradation                   | FR-013, R-001            |
| `minimumOrderValue` is null if slot not found or field absent             | Hides indicator                        | Clarification            |
| Raw response fields accessed via optional chaining with fallback defaults | Runtime safety for `unknown` response  | Clarification 2026-04-07 |
