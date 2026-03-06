---
phase: 05-products-and-bom
plan: 02
subsystem: api, database
tags: [nestjs, typeorm, bom, price-history, transactions, batch-operations]

# Dependency graph
requires:
  - phase: 05-products-and-bom
    provides: "Product entity with 5 FK dimensions, ProductsModule with CRUD"
  - phase: 04-supplies-and-price-history
    provides: "Supply entity with unitType, SupplyPriceHistory pattern"
provides:
  - "SuppliesPerProductHistory (BOM) entity with version swap pattern"
  - "ProductPriceHistory entity (append-only)"
  - "BOM CRUD endpoints with atomic transaction swap"
  - "Product price history endpoints"
  - "Batch BOM and price endpoints"
  - "Supply deactivation guard (BOM usage check)"
  - "Seed migration with 6 products, BOM entries, and selling prices"
affects: [05-03-frontend, 06-cost-calculation]

# Tech tracking
tech-stack:
  added: []
  patterns: [bom-version-swap-transaction, product-price-distinct-on, supply-deactivation-guard]

key-files:
  created:
    - nemea-back/src/products/entities/supplies-per-product-history.entity.ts
    - nemea-back/src/products/entities/product-price-history.entity.ts
    - nemea-back/src/products/dto/update-bom.dto.ts
    - nemea-back/src/products/dto/create-product-price.dto.ts
    - nemea-back/src/products/dto/batch-product-price.dto.ts
    - nemea-back/src/products/dto/batch-bom.dto.ts
    - nemea-back/src/database/migrations/1772500200000-CreateBomAndPriceHistory.ts
    - nemea-back/src/database/migrations/1772500300000-SeedProductsAndBom.ts
  modified:
    - nemea-back/src/products/products.service.ts
    - nemea-back/src/products/products.controller.ts
    - nemea-back/src/products/products.module.ts
    - nemea-back/src/supplies/supplies.service.ts
    - nemea-back/src/supplies/supplies.module.ts
    - nemea-back/.lintstagedrc.json

key-decisions:
  - "BOM version swap: deactivate old entries + insert new entries in single transaction"
  - "ProductPriceHistory append-only with DISTINCT ON for current price in findAll"
  - "Supply deactivation guard checks bomRepo.count before allowing is_active=false"
  - "Batch routes defined BEFORE :id routes to avoid NestJS UUID param collision"
  - "BatchBomDto separate from UpdateBomDto to include productIds array"
  - "lint-staged: prettier --write instead of --check to fix spawn issues on Windows"

patterns-established:
  - "BOM version swap: UPDATE is_active=false + INSERT new in transaction"
  - "Product price: append-only with DISTINCT ON batch query for current prices"
  - "Deactivation guard: cross-module repo injection for integrity checks"

requirements-completed: [PROD-05, PROD-06, PROD-07]

# Metrics
duration: 8min
completed: 2026-03-06
---

# Phase 5 Plan 2: BOM and Product Price History Summary

**BOM version swap with atomic transactions, product selling price history, supply deactivation guard, and seed data for 6 products**

## Performance

- **Duration:** 8 min
- **Started:** 2026-03-06T01:24:45Z
- **Completed:** 2026-03-06T01:32:43Z
- **Tasks:** 2
- **Files modified:** 14

## Accomplishments
- SuppliesPerProductHistory (BOM) entity with atomic version swap in transactions
- ProductPriceHistory entity with append-only pattern and DISTINCT ON current price
- 12 product endpoints total (6 existing + 6 new: BOM, prices, batch)
- Supply deactivation blocked when supply is in active BOM (409 with usage count)
- Seed migration with 6 realistic products, BOM entries, supply prices, and selling prices

## Task Commits

Each task was committed atomically:

1. **Task 1: BOM and ProductPriceHistory entities + migration + endpoints** - `fa42554` (feat)
2. **Task 2: Supply deactivation guard + seed migration** - `5a5db28` (feat)

## Files Created/Modified
- `nemea-back/src/products/entities/supplies-per-product-history.entity.ts` - BOM entity with product/supply FK, quantity, is_active
- `nemea-back/src/products/entities/product-price-history.entity.ts` - Price history with decimal price, currency
- `nemea-back/src/products/dto/update-bom.dto.ts` - BOM items array with ValidateNested
- `nemea-back/src/products/dto/create-product-price.dto.ts` - Single product price DTO
- `nemea-back/src/products/dto/batch-product-price.dto.ts` - Batch price with productIds array
- `nemea-back/src/products/dto/batch-bom.dto.ts` - Batch BOM with productIds + items
- `nemea-back/src/products/products.service.ts` - Added BOM CRUD, price history, batch methods, ProductWithPrice
- `nemea-back/src/products/products.controller.ts` - 6 new endpoints (BOM + price + batch)
- `nemea-back/src/products/products.module.ts` - Added BOM, PriceHistory, Supply to TypeORM
- `nemea-back/src/supplies/supplies.service.ts` - BOM usage guard on toggleStatus
- `nemea-back/src/supplies/supplies.module.ts` - Added SuppliesPerProductHistory import
- `nemea-back/src/database/migrations/1772500200000-CreateBomAndPriceHistory.ts` - Tables + indexes
- `nemea-back/src/database/migrations/1772500300000-SeedProductsAndBom.ts` - Seed products, BOM, prices

## Decisions Made
- BOM version swap uses EntityManager.transaction for atomicity (old entries is_active=false, new entries is_active=true)
- ProductPriceHistory is append-only; current price resolved via DISTINCT ON in findAll
- Supply deactivation guard uses cross-module repo injection (SuppliesPerProductHistory injected into SuppliesService)
- Batch routes (batch-bom, batch-prices) defined before :id routes in controller to prevent NestJS treating them as UUID params
- Created separate BatchBomDto (with productIds) rather than reusing UpdateBomDto for the batch endpoint
- lint-staged changed from prettier --check to prettier --write to resolve Windows spawn failures

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed lint-staged prettier spawn on Windows**
- **Found during:** Task 1 (commit phase)
- **Issue:** lint-staged with `prettier --check` failed to spawn on Windows/MINGW64 for TS files
- **Fix:** Changed to `prettier --write` which works reliably and auto-formats on commit
- **Files modified:** nemea-back/.lintstagedrc.json
- **Verification:** Commit hooks pass successfully
- **Committed in:** fa42554 (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Pre-existing config issue blocking commits. Fix was minimal and necessary.

## Issues Encountered
None beyond the lint-staged issue documented above.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- BOM and price history API complete, ready for frontend product management UI (plan 05-03)
- ProductsService with ProductWithPrice type ready for cost calculation module (phase 06)
- Supply deactivation guard ensures BOM data integrity

## Self-Check: PASSED
