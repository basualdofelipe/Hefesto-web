---
phase: 06-cost-calculation
plan: 02
subsystem: ui
tags: [nextjs, react, cost-display, margin-calculation, product-detail]

requires:
  - phase: 06-cost-calculation
    provides: CostsModule with enriched products API returning cost, costBreakdown, costWarnings
  - phase: 05-products-and-bom
    provides: Product entity, BOM expanded row, ProductTypeGroup, ProductTable

provides:
  - Product list with Costo and Margen columns (8-column layout)
  - Enriched BOM expanded row with Precio Unit. and Costo Linea columns, total row, margin summary
  - Product detail page at /productos/:id with full cost breakdown
  - Group header average cost display
  - Product name links from list to detail page

affects: [07-expenses-and-config, frontend-products]

tech-stack:
  added: []
  patterns: [cost/margin formatting helpers, IIFE pattern for inline computed JSX, parallel fetch for detail+BOM data]

key-files:
  created:
    - nemea-front/src/app/(app)/productos/[id]/page.tsx
    - nemea-front/src/components/products/ProductDetailClient.tsx
  modified:
    - nemea-front/src/components/products/types.ts
    - nemea-front/src/components/products/ProductTypeGroup.tsx
    - nemea-front/src/components/products/ProductExpandedRow.tsx

key-decisions:
  - "Expanded row fetches product detail in parallel with BOM to get costBreakdown data"
  - "formatCost uses es-AR locale with 0-2 decimal fraction digits for consistent currency display"
  - "formatMargin calculates markup percentage (margin/cost) not gross margin (margin/price)"
  - "Product name in list is a Link to detail page with stopPropagation to avoid row expand"

patterns-established:
  - "Cost formatting: formatCost(null) returns em-dash, formatCost(number) returns $-prefixed es-AR locale string"
  - "Margin display: dollar amount + percentage in parentheses as smaller muted text"
  - "Cost warnings: AlertTriangle icon with Tooltip listing all warnings"

requirements-completed: [COST-02, COST-03, COST-04]

duration: 6min
completed: 2026-03-06
---

# Phase 6 Plan 2: Frontend Cost Display Summary

**Product list with Costo/Margen columns, enriched BOM table with line costs, and product detail page at /productos/:id with full cost breakdown and margin summary**

## Performance

- **Duration:** 6 min
- **Started:** 2026-03-06T21:33:54Z
- **Completed:** 2026-03-06T21:40:00Z
- **Tasks:** 2 of 2 auto tasks (Task 3 is checkpoint:human-verify)
- **Files modified:** 5

## Accomplishments
- Product list table expanded to 8 columns: SKU, Nombre, Terminacion, Color, Talle, Costo, Precio Venta, Margen
- Enriched BOM expanded row with Precio Unit. and Costo Linea columns, total row, and margin summary
- Product detail page at /productos/:id with header, BOM cost breakdown, margin summary card, and admin actions
- Group headers show average cost badge excluding products without BOM
- Product names link to detail page from the list

## Task Commits

Each task was committed atomically:

1. **Task 1: Extend types + cost/margin columns in product list + group header aggregation** - `61e9dcd` (feat)
2. **Task 2: Product detail page at /productos/:id** - `9d3d9fe` (feat)

## Files Created/Modified
- `nemea-front/src/components/products/types.ts` - Added CostBreakdownItem interface, formatCost, formatMargin helpers, extended Product with cost fields
- `nemea-front/src/components/products/ProductTypeGroup.tsx` - 8-column layout, Costo/Margen cells, tooltips, warnings, name links, avg cost badge
- `nemea-front/src/components/products/ProductExpandedRow.tsx` - Enriched BOM table with cost columns, total row, margin summary, parallel fetch for cost data
- `nemea-front/src/app/(app)/productos/[id]/page.tsx` - Server component fetching product+BOM+catalogs+supplies
- `nemea-front/src/components/products/ProductDetailClient.tsx` - Detail page client component with BOM breakdown, margin summary, admin actions

## Decisions Made
- Expanded row fetches product detail (`GET /api/products/:id`) in parallel with BOM fetch to get costBreakdown data, since list endpoint returns costBreakdown as null for performance
- formatMargin calculates markup percentage (margin/cost * 100) not gross margin (margin/price * 100), matching typical leatherwork business convention
- Product name cell uses Next.js Link with stopPropagation to prevent row expand when navigating to detail page
- Cost formatting uses es-AR locale for Argentine peso display with 0-2 decimal digits

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Task 3 (checkpoint:human-verify) pending for visual verification
- All frontend code compiles without type errors
- All 39 backend tests pass
- Ready for Phase 7 (Expenses & Config) after verification

## Self-Check: PASSED

- All 5 files verified on disk
- Both commits verified in git log (61e9dcd, 9d3d9fe)
- TypeScript compilation clean (npx tsc --noEmit)
- All 39 backend tests pass (npx jest --no-coverage)

---
*Phase: 06-cost-calculation*
*Completed: 2026-03-06*
