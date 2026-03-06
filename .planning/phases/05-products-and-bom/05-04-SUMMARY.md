---
phase: 05-products-and-bom
plan: 04
subsystem: ui
tags: [react, shadcn, combobox, bom, pricing, batch-operations, dialog]

requires:
  - phase: 05-03
    provides: "Product table with grouped display, batch create, edit, BOM display stubs"
  - phase: 05-02
    provides: "BOM and price history backend endpoints"
  - phase: 04-02
    provides: "Supply types, AddPriceInline and PriceHistoryDialog patterns"
provides:
  - "BomEditorDialog with supply Combobox grouped by type"
  - "BomGroupEditorDialog with majority BOM detection and product checkboxes"
  - "AddSellingPriceInline for inline price entry"
  - "PriceHistoryDialog for product selling price history"
  - "BatchPriceDialog for group price updates"
  - "All product action buttons wired in expanded row and group header"
affects: [06-cost-calculation]

tech-stack:
  added: []
  patterns: ["majority-detection via serialized JSON comparison", "batch operations with checkbox selection"]

key-files:
  created:
    - nemea-front/src/components/products/BomEditorDialog.tsx
    - nemea-front/src/components/products/BomGroupEditorDialog.tsx
    - nemea-front/src/components/products/AddSellingPriceInline.tsx
    - nemea-front/src/components/products/PriceHistoryDialog.tsx
    - nemea-front/src/components/products/BatchPriceDialog.tsx
  modified:
    - nemea-front/src/components/products/ProductExpandedRow.tsx
    - nemea-front/src/components/products/ProductTypeGroup.tsx
    - nemea-back/src/database/migrations/1772500000000-AddSkuCodeToCatalogs.ts

key-decisions:
  - "BomGroupEditorDialog uses majority BOM detection via serialized JSON comparison to identify differing products"
  - "Migration fix: dynamic ROW_NUMBER fallback for user-created catalog items not covered by CASE statements"
  - "Auth ClientFetchError is pre-existing (not phase 5), documented but not fixed"

patterns-established:
  - "Majority detection: serialize items as sorted JSON, find most common, warn on differences"
  - "Batch dialog: checkbox list with select-all, single input, POST to batch endpoint"

requirements-completed: [PROD-05, PROD-06, PROD-07]

duration: 12min
completed: 2026-03-06
---

# Phase 5 Plan 04: BOM/Price Frontend Wiring Summary

**BOM editor with supply Combobox, group BOM editor with majority detection, selling price inline/history/batch, and migration fix for user-created catalog items**

## Performance

- **Duration:** 12 min
- **Started:** 2026-03-06T01:42:00Z
- **Completed:** 2026-03-06T01:54:00Z
- **Tasks:** 3 (2 auto + 1 continuation fix)
- **Files modified:** 8

## Accomplishments
- BomEditorDialog with supply Combobox grouped by supply type, quantity input, unit auto-fill
- BomGroupEditorDialog with product checkboxes, majority BOM detection, warning indicators
- AddSellingPriceInline, PriceHistoryDialog, BatchPriceDialog completing all price management UX
- All action buttons wired in ProductExpandedRow and ProductTypeGroup headers
- Fixed blocking migration bug: sku_code CASE statements now handle user-created catalog items

## Task Commits

Each task was committed atomically:

1. **Task 1: BomEditorDialog + BomGroupEditorDialog + wire group header buttons** - `0000f8a` (feat)
2. **Task 2: AddSellingPriceInline + PriceHistoryDialog + BatchPriceDialog + wire remaining buttons** - `1215418` (feat)
3. **Task 3 (bug fix): Fix sku_code migration for user-created catalog items** - `ddf12b9` (fix)

## Files Created/Modified
- `nemea-front/src/components/products/BomEditorDialog.tsx` - BOM editor modal with supply Combobox
- `nemea-front/src/components/products/BomGroupEditorDialog.tsx` - Group BOM editor with majority detection
- `nemea-front/src/components/products/AddSellingPriceInline.tsx` - Inline selling price input
- `nemea-front/src/components/products/PriceHistoryDialog.tsx` - Price history dialog
- `nemea-front/src/components/products/BatchPriceDialog.tsx` - Batch price update dialog
- `nemea-front/src/components/products/ProductExpandedRow.tsx` - Wired all action buttons
- `nemea-front/src/components/products/ProductTypeGroup.tsx` - Wired group header buttons
- `nemea-back/src/database/migrations/1772500000000-AddSkuCodeToCatalogs.ts` - Fixed sku_code assignment for dynamic catalog items

## Decisions Made
- BomGroupEditorDialog uses majority BOM detection by serializing each product's BOM as sorted JSON and finding the most common pattern
- Migration fix uses ROW_NUMBER() window function to dynamically assign sku_codes to any catalog items not covered by the hardcoded CASE statements
- Auth ClientFetchError on homepage confirmed as pre-existing issue (NextAuth session endpoint returning HTML instead of JSON) -- no auth files modified in phase 5

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed sku_code migration failing on user-created catalog items**
- **Found during:** Task 3 (continuation -- reported by user)
- **Issue:** Migration `1772500000000-AddSkuCodeToCatalogs.ts` CASE statements only cover seed items from phase 3. User-created catalog items via the UI have no CASE match, remain NULL, then SET NOT NULL fails.
- **Fix:** Added dynamic ROW_NUMBER() fallback that assigns sequential sku_codes (continuing from max existing) to any rows still NULL after the CASE updates.
- **Files modified:** `nemea-back/src/database/migrations/1772500000000-AddSkuCodeToCatalogs.ts`
- **Verification:** Backend TypeScript compiles clean (`npx tsc --noEmit`)
- **Committed in:** `ddf12b9`

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential fix for backend startup. No scope creep.

## Issues Encountered
- Auth ClientFetchError on frontend homepage: NextAuth session endpoint returns HTML instead of JSON. Confirmed pre-existing (no auth files modified in phase 5). Not fixed -- out of scope for this plan. Noted for future investigation.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Phase 5 (Products and BOM) is complete: all 4 plans executed
- Full product lifecycle functional: batch create, edit, BOM management, price tracking
- Ready for Phase 6 (Cost Calculation) which depends on products + BOM + supply prices
- Pre-existing auth issue should be investigated before Phase 6 if it blocks testing

## Self-Check: PASSED

- All 5 created files exist
- All 3 commits verified (0000f8a, 1215418, ddf12b9)
- Frontend: tsc --noEmit clean, lint 0 errors (1 pre-existing warning)
- Backend: tsc --noEmit clean

---
*Phase: 05-products-and-bom*
*Completed: 2026-03-06*
