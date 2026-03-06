---
phase: 05-products-and-bom
plan: 01
subsystem: api, database
tags: [nestjs, typeorm, products, sku, batch-creation, migrations]

# Dependency graph
requires:
  - phase: 03-catalogs-and-suppliers
    provides: "5 catalog dimension entities (ProductType, ProductName, ProductFinish, ProductColor, ProductSize)"
provides:
  - "Product entity with 5 FK dimensions + skuCode + isActive"
  - "ProductsModule with CRUD + batch creation + SKU auto-generation"
  - "2 migrations: sku_code on catalogs + products table with partial unique index"
  - "6 REST endpoints for product management"
affects: [05-02-bom, 06-cost-calculation]

# Tech tracking
tech-stack:
  added: []
  patterns: [sku-generation-from-dimensions, batch-cartesian-product, partial-unique-index]

key-files:
  created:
    - nemea-back/src/products/entities/product.entity.ts
    - nemea-back/src/products/products.service.ts
    - nemea-back/src/products/products.controller.ts
    - nemea-back/src/products/products.module.ts
    - nemea-back/src/products/dto/create-product.dto.ts
    - nemea-back/src/products/dto/create-batch-products.dto.ts
    - nemea-back/src/products/dto/update-product.dto.ts
    - nemea-back/src/database/migrations/1772500000000-AddSkuCodeToCatalogs.ts
    - nemea-back/src/database/migrations/1772500100000-CreateProductsTable.ts
  modified:
    - nemea-back/src/catalogs/entities/product-type.entity.ts
    - nemea-back/src/catalogs/entities/product-name.entity.ts
    - nemea-back/src/catalogs/entities/product-finish.entity.ts
    - nemea-back/src/catalogs/entities/product-color.entity.ts
    - nemea-back/src/catalogs/entities/product-size.entity.ts
    - nemea-back/src/catalogs/dto/create-catalog-item.dto.ts
    - nemea-back/src/app.module.ts
    - nemea-back/.lintstagedrc.json

key-decisions:
  - "SKU format: type.name.finish.color.size using skuCode SMALLINT from each dimension"
  - "Partial unique index on products.sku_code WHERE is_active = true (same pattern as supplies)"
  - "Batch creation via entityManager.transaction with cartesian product of colors x sizes"
  - "UpdateProductDto requires all 5 dimension IDs (SKU regenerates on any change)"
  - "ProductsService exported for future BOM and cost calculation modules"

patterns-established:
  - "SKU generation: dot-separated numeric codes from dimension entities"
  - "Batch creation: load shared dimensions once, nested loop for cartesian, single transaction"
  - "loadDimensions helper: private method to validate and load all 5 FK entities"

requirements-completed: [PROD-01, PROD-02, PROD-03, PROD-04]

# Metrics
duration: 5min
completed: 2026-03-06
---

# Phase 5 Plan 1: Products Entity and CRUD Summary

**Product entity with 5-dimension SKU auto-generation, batch creation via cartesian product, and 6 REST endpoints**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-06T01:16:26Z
- **Completed:** 2026-03-06T01:21:47Z
- **Tasks:** 2
- **Files modified:** 17

## Accomplishments
- Product entity with ManyToOne relations to all 5 catalog dimensions (type, name, finish, color, size)
- SKU auto-generation from dimension sku_codes (e.g. 1.2.1.3.0)
- Batch product creation from cartesian product of colors x sizes in a single transaction
- Partial unique index ensures no duplicate active SKUs while allowing reactivation
- sku_code SMALLINT column added to all 5 catalog dimension tables with seeded values

## Task Commits

Each task was committed atomically:

1. **Task 1: sku_code migration + catalog entity/DTO updates + products table migration** - `b6a2b56` (feat)
2. **Task 2: ProductsModule with CRUD + batch creation + SKU generation + toggle-status** - `ae20411` (feat)

## Files Created/Modified
- `nemea-back/src/products/entities/product.entity.ts` - Product entity with 5 FK dimensions, skuCode, isActive
- `nemea-back/src/products/products.service.ts` - CRUD + batch + SKU generation + toggle-status
- `nemea-back/src/products/products.controller.ts` - 6 REST endpoints with Swagger decorators
- `nemea-back/src/products/products.module.ts` - Module with TypeORM repos, exports ProductsService
- `nemea-back/src/products/dto/create-product.dto.ts` - DTO with 5 UUID dimension IDs
- `nemea-back/src/products/dto/create-batch-products.dto.ts` - DTO with colorIds[] and sizeIds[] arrays
- `nemea-back/src/products/dto/update-product.dto.ts` - Same as create (all required, SKU regenerates)
- `nemea-back/src/database/migrations/1772500000000-AddSkuCodeToCatalogs.ts` - Adds sku_code to 5 catalog tables
- `nemea-back/src/database/migrations/1772500100000-CreateProductsTable.ts` - Products table with partial unique index
- `nemea-back/src/catalogs/entities/*.entity.ts` - All 5 dimension entities now have skuCode field
- `nemea-back/src/catalogs/dto/create-catalog-item.dto.ts` - Optional skuCode field added
- `nemea-back/src/app.module.ts` - ProductsModule registered

## Decisions Made
- SKU format uses dot-separated SMALLINT codes from each dimension (e.g. 1.2.1.3.0)
- Partial unique index on products.sku_code WHERE is_active = true (consistent with supplies pattern)
- Batch creation uses entityManager.transaction for atomicity
- UpdateProductDto requires all 5 fields (not partial) since SKU must regenerate completely
- product_sizes: "Unico" gets sku_code=0 (sentinel for single-size products)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed lint-staged prettier spawn on Windows**
- **Found during:** Task 1 (commit phase)
- **Issue:** lint-staged config called `prettier --check` but prettier binary not in PATH on Windows/MINGW64
- **Fix:** Changed `.lintstagedrc.json` to use `npx prettier --check` for both *.ts and *.json rules
- **Files modified:** nemea-back/.lintstagedrc.json
- **Verification:** Commit hooks pass successfully
- **Committed in:** b6a2b56 (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Pre-existing config issue blocking all commits. Fix was minimal and necessary.

## Issues Encountered
None beyond the lint-staged issue documented above.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Product entity and CRUD API ready for BOM (Bill of Materials) in plan 05-02
- ProductsService exported for injection into future BOMModule and CostsModule
- SKU system established, ready for frontend product management UI

## Self-Check: PASSED

All 9 created files verified on disk. Both task commits (b6a2b56, ae20411) found in git log.

---
*Phase: 05-products-and-bom*
*Completed: 2026-03-06*
