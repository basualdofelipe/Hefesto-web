---
phase: 03-catalogs-and-suppliers
plan: 02
subsystem: api
tags: [nestjs, typeorm, crud, catalogs, suppliers, uuid, swagger, soft-delete]

# Dependency graph
requires:
  - phase: 03-catalogs-and-suppliers
    provides: "UUID-based BaseEntity, users table migrated to UUID"
provides:
  - "CatalogsModule with generic CRUD for 6 product dimension tables"
  - "SuppliersModule with full CRUD + toggle-status soft delete"
  - "7 database tables: product_types, product_names, product_finishes, product_colors, product_sizes, supply_types, suppliers"
  - "Seed data for all catalog dimensions and suppliers from business Excel"
  - "Partial unique index on supplier name (active only)"
  - "FK-safe delete with 409 ConflictException for future foreign key references"
affects: [04-supplies-and-price-history, 05-products-and-bom, 06-cost-calculation]

# Tech tracking
tech-stack:
  added: []
  patterns: [generic-dimension-routing, partial-unique-index, soft-delete-toggle]

key-files:
  created:
    - nemea-back/src/catalogs/entities/product-type.entity.ts
    - nemea-back/src/catalogs/entities/product-name.entity.ts
    - nemea-back/src/catalogs/entities/product-finish.entity.ts
    - nemea-back/src/catalogs/entities/product-color.entity.ts
    - nemea-back/src/catalogs/entities/product-size.entity.ts
    - nemea-back/src/catalogs/entities/supply-type.entity.ts
    - nemea-back/src/suppliers/entities/supplier.entity.ts
    - nemea-back/src/catalogs/catalogs.module.ts
    - nemea-back/src/catalogs/catalogs.controller.ts
    - nemea-back/src/catalogs/catalogs.service.ts
    - nemea-back/src/catalogs/dto/create-catalog-item.dto.ts
    - nemea-back/src/catalogs/dto/update-catalog-item.dto.ts
    - nemea-back/src/suppliers/suppliers.module.ts
    - nemea-back/src/suppliers/suppliers.controller.ts
    - nemea-back/src/suppliers/suppliers.service.ts
    - nemea-back/src/suppliers/dto/create-supplier.dto.ts
    - nemea-back/src/suppliers/dto/update-supplier.dto.ts
    - nemea-back/src/database/migrations/1772380782730-CreateCatalogAndSupplierTables.ts
    - nemea-back/src/database/migrations/1772380800000-SeedCatalogsAndSuppliers.ts
  modified:
    - nemea-back/src/app.module.ts

key-decisions:
  - "Generic dimension-to-repository map in CatalogsService avoids 6 duplicate controllers for identical name-only entities"
  - "Partial unique index on suppliers (WHERE is_active = true) allows reactivating deactivated supplier names"
  - "FK-safe delete prepared with 23503 catch for future Phase 5 product references"
  - "Supplier toggle-status via PATCH endpoint instead of DELETE for soft delete"

patterns-established:
  - "Generic catalog CRUD: Single controller handles all name-only dimension tables via :dimension path param"
  - "Soft delete toggle: PATCH /resource/:id/toggle-status flips isActive boolean"
  - "Partial unique index: Unique constraint scoped to active records only"
  - "Idempotent seed migration: INSERT ... ON CONFLICT DO NOTHING"

requirements-completed: [CATL-01, CATL-02, CATL-03, CATL-04, CATL-05, CATL-06, SUPP-01, SUPP-02]

# Metrics
duration: 5min
completed: 2026-03-01
---

# Phase 3 Plan 2: Catalogs and Suppliers API Summary

**Generic CRUD for 6 product dimension catalogs via :dimension routing + SuppliersModule with full CRUD and toggle-status soft delete, 7 tables with UUID PKs and idempotent seed data**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-01T15:59:09Z
- **Completed:** 2026-03-01T16:04:06Z
- **Tasks:** 3
- **Files modified:** 20

## Accomplishments
- Created 7 entities (6 catalog dimensions + 1 supplier) all extending BaseEntity with UUID PKs
- Built CatalogsModule with a generic dimension-to-repository map avoiding code duplication for 6 identical tables
- Built SuppliersModule with full CRUD + PATCH toggle-status for soft delete
- Created migrations (auto-generated schema + idempotent seed data from business Excel)
- Both modules registered in AppModule, server boots cleanly, Swagger docs show all endpoints
- All 31 existing tests still pass

## Task Commits

Each task was committed atomically:

1. **Task 1: Create catalog and supplier entities + migrations** - `c4f8476` (feat) [nemea-back]
2. **Task 2: Create CatalogsModule with generic CRUD controller and service** - `92d7a87` (feat) [nemea-back]
3. **Task 3: Create SuppliersModule with CRUD + toggle status + register both modules** - `21878c6` (feat) [nemea-back]

## Files Created/Modified
- `nemea-back/src/catalogs/entities/product-type.entity.ts` - ProductType entity (name varchar 100, unique)
- `nemea-back/src/catalogs/entities/product-name.entity.ts` - ProductName entity
- `nemea-back/src/catalogs/entities/product-finish.entity.ts` - ProductFinish entity
- `nemea-back/src/catalogs/entities/product-color.entity.ts` - ProductColor entity
- `nemea-back/src/catalogs/entities/product-size.entity.ts` - ProductSize entity
- `nemea-back/src/catalogs/entities/supply-type.entity.ts` - SupplyType entity
- `nemea-back/src/suppliers/entities/supplier.entity.ts` - Supplier entity with partial unique index on name (active only)
- `nemea-back/src/catalogs/catalogs.module.ts` - CatalogsModule (imports 6 entity repos, exports CatalogsService)
- `nemea-back/src/catalogs/catalogs.controller.ts` - Generic CRUD controller with :dimension path param
- `nemea-back/src/catalogs/catalogs.service.ts` - Dimension-to-repository map, FK-safe delete, unique constraint handling
- `nemea-back/src/catalogs/dto/create-catalog-item.dto.ts` - Shared DTO for all 6 catalog dimensions
- `nemea-back/src/catalogs/dto/update-catalog-item.dto.ts` - PartialType of CreateCatalogItemDto
- `nemea-back/src/suppliers/suppliers.module.ts` - SuppliersModule (exports SuppliersService)
- `nemea-back/src/suppliers/suppliers.controller.ts` - CRUD + toggle-status with Swagger docs
- `nemea-back/src/suppliers/suppliers.service.ts` - Full CRUD with findAll/findActive, toggle, FK-safe delete
- `nemea-back/src/suppliers/dto/create-supplier.dto.ts` - Supplier DTO with email validation
- `nemea-back/src/suppliers/dto/update-supplier.dto.ts` - PartialType of CreateSupplierDto
- `nemea-back/src/database/migrations/1772380782730-CreateCatalogAndSupplierTables.ts` - Schema migration for 7 tables
- `nemea-back/src/database/migrations/1772380800000-SeedCatalogsAndSuppliers.ts` - Idempotent seed data
- `nemea-back/src/app.module.ts` - Added CatalogsModule and SuppliersModule to imports

## Decisions Made
- Used a generic dimension-to-repository map in CatalogsService to avoid duplicating 6 identical controller/service pairs for name-only catalog tables
- Partial unique index on suppliers (`WHERE is_active = true`) instead of `@Unique(['name'])` to allow reactivating deactivated supplier names (per Phase 3 research Pitfall 6)
- FK-safe delete in both CatalogsService and SuppliersService catches PostgreSQL error code 23503 and returns 409 ConflictException -- no FK references exist yet in Phase 3 but this prepares for Phase 5 when products reference catalog dimensions
- PATCH toggle-status endpoint for supplier soft delete instead of using `@DeleteDateColumn` -- simpler pattern that matches the UI toggle switch interaction

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All 6 catalog dimension tables populated with real business data
- Suppliers table seeded with 3 initial suppliers
- CatalogsService exported for use by future SuppliesModule (supply_type validation)
- SuppliersService exported for use by future SuppliesModule (supplier validation)
- Ready for Plan 03-03 (frontend catalog/supplier pages) or Phase 4 (supplies)

## Self-Check: PASSED

- FOUND: All 20 files (6 catalog entities, 1 supplier entity, 2 modules, 2 controllers, 2 services, 4 DTOs, 2 migrations, 1 app.module.ts)
- FOUND: commit c4f8476 (Task 1 entities + migrations)
- FOUND: commit 92d7a87 (Task 2 CatalogsModule)
- FOUND: commit 21878c6 (Task 3 SuppliersModule + AppModule)
- TypeScript: 0 errors
- Tests: 31/31 passing
- Server: boots successfully, all routes mapped

---
*Phase: 03-catalogs-and-suppliers*
*Completed: 2026-03-01*
