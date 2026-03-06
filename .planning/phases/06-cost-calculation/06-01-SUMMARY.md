---
phase: 06-cost-calculation
plan: 01
subsystem: api
tags: [nestjs, typeorm, cost-calculation, distinct-on, batch-query]

requires:
  - phase: 05-products-and-bom
    provides: Product entity, SuppliesPerProductHistory BOM entity, ProductPriceHistory, ProductsService
  - phase: 04-supplies-and-price-history
    provides: Supply entity, SupplyPriceHistory entity with DISTINCT ON price pattern

provides:
  - CostsModule with CostsService for batched 2-query cost calculation
  - ProductWithCost interface enriching products API with cost, costBreakdown, costWarnings
  - calculateAll() returns Map<string, ProductCostData> for all products
  - calculateForProduct(id) returns single product cost with full breakdown

affects: [06-cost-calculation, frontend-products]

tech-stack:
  added: []
  patterns: [2-query batch cost calculation, DISTINCT ON for latest prices, in-memory grouping]

key-files:
  created:
    - nemea-back/src/costs/costs.module.ts
    - nemea-back/src/costs/costs.service.ts
    - nemea-back/src/costs/costs.service.spec.ts
    - nemea-back/src/costs/dto/product-with-cost.dto.ts
  modified:
    - nemea-back/src/app.module.ts
    - nemea-back/src/products/products.module.ts
    - nemea-back/src/products/products.service.ts
    - nemea-back/src/products/products.controller.ts

key-decisions:
  - "CostsModule imports only entity repos (no ProductsModule/SuppliesModule) to avoid circular deps"
  - "Cost breakdown excluded from findAll list response (null) for performance, included only in findOne detail"
  - "findOneWithPrice added to ProductsService for single-product price fetch in detail view"

patterns-established:
  - "2-query cost pattern: BOM find + DISTINCT ON prices, in-memory join and grouping"
  - "ProductWithCost extends ProductWithPrice with cost/costBreakdown/costWarnings"

requirements-completed: [COST-01, COST-02, COST-03]

duration: 5min
completed: 2026-03-06
---

# Phase 6 Plan 1: Cost Calculation Summary

**Batched 2-query cost calculation via CostsService with enriched products API returning cost, breakdown, and warnings**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-06T21:25:51Z
- **Completed:** 2026-03-06T21:30:24Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments
- CostsService with calculateAll() and calculateForProduct() using exactly 2 SQL queries (no N+1)
- GET /products returns cost and costWarnings per product
- GET /products/:id returns full costBreakdown with per-supply line costs
- Products without BOM return cost: null; supplies without prices generate warnings
- 8 unit tests covering all cost scenarios (rounding, missing prices, inactive supplies)

## Task Commits

Each task was committed atomically:

1. **Task 1: CostsModule + CostsService with batched cost calculation** - `0978a9e` (test) + `f482173` (feat)
2. **Task 2: Enrich products endpoints with cost data** - `215c5d1` (feat)

## Files Created/Modified
- `nemea-back/src/costs/dto/product-with-cost.dto.ts` - CostBreakdownItem, ProductCostData, ProductWithCost interfaces
- `nemea-back/src/costs/costs.service.ts` - 2-query batch cost calculation engine
- `nemea-back/src/costs/costs.module.ts` - Module with BOM and price history entity repos
- `nemea-back/src/costs/costs.service.spec.ts` - 8 unit tests for all cost scenarios
- `nemea-back/src/app.module.ts` - Added CostsModule to imports
- `nemea-back/src/products/products.module.ts` - Added CostsModule import for DI
- `nemea-back/src/products/products.service.ts` - Added findOneWithPrice method
- `nemea-back/src/products/products.controller.ts` - Enriched findAll/findOne with cost data

## Decisions Made
- CostsModule imports only TypeORM entity repos (SuppliesPerProductHistory, SupplyPriceHistory) directly, avoiding circular dependency with ProductsModule and SuppliesModule
- Cost breakdown excluded from list endpoint (set to null) for performance; full breakdown only in detail view
- findOneWithPrice method added to ProductsService to fetch single product with latest selling price for the detail endpoint

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Cost calculation backend complete and tested
- Ready for Plan 06-02 (frontend cost display integration)
- All 39 backend tests pass, no type errors

## Self-Check: PASSED

- All 4 created files verified on disk
- All 3 commits verified in git log (0978a9e, f482173, 215c5d1)
- TypeScript compilation clean (npx tsc --noEmit)
- All 39 tests pass (npx jest --no-coverage)

---
*Phase: 06-cost-calculation*
*Completed: 2026-03-06*
