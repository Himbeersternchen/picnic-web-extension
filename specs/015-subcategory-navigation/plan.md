# Implementation Plan: Sub-category Navigation

**Branch**: `015-subcategory-navigation` | **Date**: 2026-04-16 | **Spec**: `specs/015-subcategory-navigation/spec.md`
**Input**: Feature specification from `/specs/015-subcategory-navigation/spec.md`

## Summary

Enable drill-down navigation from top-level categories to their sub-categories. When a user taps a top-level category, fetch its L1 sub-categories from `client.app.getPage("L1-category-page-root?category_id={id}")` and display them in the same list-row layout. Research confirmed the L1 sub-category PML items use the identical structure as top-level categories (`core-list-item-category-{id}` with `accessibilityLabel`, `onPress.target`, IMAGE source). L2 pages contain products (out of scope) — leaf sub-category taps are no-ops.

## Technical Context

**Language/Version**: TypeScript 5, Node.js 20.9+
**Primary Dependencies**: Next.js 16.2.1 (App Router), React 19.2.4, Tailwind CSS 4, picnic-api ^4.1.0
**Storage**: N/A (no persistent storage; navigation state is ephemeral client-side)
**Testing**: `npm run lint && npm run build` (no test framework configured)
**Target Platform**: Web (mobile-first responsive)
**Project Type**: Web application (Next.js App Router)
**Performance Goals**: Sub-category fetch + render within 2 seconds (SC-001)
**Constraints**: All files < 300 lines, max 3 levels nesting
**Scale/Scope**: 26 top-level categories, ~5-10 sub-categories per parent

## Constitution Check

_GATE: All principles verified._

| Principle                    | Status | Notes                                                                                                                        |
| ---------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------- |
| I. SRP/DRY/DI                | PASS   | Parser logic shared via extracted helper; new route has single responsibility; client injected via `buildPicnicClient`       |
| II. Naming                   | PASS   | `parseSubcategoryPage`, `extractPageTitle`, `SubcategoriesApiResponse`, `CategoryNavState` — all follow conventions          |
| III. Forbidden Anti-Patterns | PASS   | No file exceeds 300 lines; no deep nesting; constants extracted (`L1_PAGE_PREFIX`, `CATEGORY_ITEM_PREFIX`); no empty catches |
| IV. Self-Refactor            | PASS   | Will enforce at implementation time                                                                                          |
| V. Readability               | PASS   | Linear control flow with early returns; explicit types; no clever constructs                                                 |

## Project Structure

### Documentation (this feature)

```text
specs/015-subcategory-navigation/
├── plan.md              # This file
├── research.md          # Phase 0 output — L1/L2 API response analysis
├── data-model.md        # Phase 1 output — types and field mappings
├── quickstart.md        # Phase 1 output — implementation guide
├── contracts/
│   └── subcategories-api.md  # Phase 1 output — new API endpoint contract
├── checklists/
│   └── requirements.md  # Quality checklist (from specify phase)
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
src/
├── lib/
│   ├── category-types.ts          # MODIFY — add SubcategoriesApiResponse
│   ├── parse-categories.ts        # MODIFY — extract shared helper
│   ├── parse-subcategories.ts     # NEW — L1 sub-category parser
│   └── pml-helpers.ts             # UNCHANGED (reused)
├── components/
│   ├── category-grid.tsx          # MODIFY — add onCategoryTap callback
│   └── shortcut-list.tsx          # UNCHANGED
├── app/
│   ├── page.tsx                   # MODIFY — add navigation state + sub-category view
│   └── api/
│       └── categories/
│           ├── route.ts           # UNCHANGED
│           └── [categoryId]/
│               └── subcategories/
│                   └── route.ts   # NEW — GET /api/categories/[categoryId]/subcategories
```

**Structure Decision**: Follows existing Next.js App Router convention with dynamic route segments. New files are minimal (1 parser, 1 route). Shared extraction logic is DRY'd between `parse-categories.ts` and `parse-subcategories.ts`.

## Complexity Tracking

No constitution violations. All new files well under 300-line limit. No new dependencies.
