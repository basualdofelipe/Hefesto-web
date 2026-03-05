---
phase: 04-supplies-and-price-history
plan: 01
subsystem: api
tags: [nestjs, typeorm, crud, supplies, price-history, decimal, soft-delete, cascade, swagger]

# Dependency graph
requires:
  - phase: 03-catalogs-and-suppliers
    provides: "SupplyType entity, Supplier entity with is_active, SuppliersModule with toggleStatus"
provides:
  - "Supply entity with ManyToOne to SupplyType and Supplier, UnitType enum, is_active with partial unique index"
  - "SupplyPriceHistory entity with append-only price records and composite index"
  - "SuppliesModule with full CRUD + toggle-status + price history endpoints"
  - "Supplier cascade deactivation (deactivate supplier -> deactivate its active supplies)"
  - "2-query DISTINCT ON pattern for efficient current price retrieval"
affects: [04-02-frontend, 05-products-and-bom, 06-cost-calculation]

# Tech tracking
tech-stack:
  added: []
  patterns: [append-only-price-history, transactional-create-with-price, distinct-on-latest-price, cascade-deactivation]

key-files:
  created:
    - nemea-back/src/supplies/entities/supply.entity.ts
    - nemea-back/src/supplies/entities/supply-price-history.entity.ts
    - nemea-back/src/supplies/dto/create-supply.dto.ts
    - nemea-back/src/supplies/dto/update-supply.dto.ts
    - nemea-back/src/supplies/dto/create-supply-price.dto.ts
    - nemea-back/src/supplies/supplies.service.ts
    - nemea-back/src/supplies/supplies.controller.ts
    - nemea-back/src/supplies/supplies.module.ts
    - nemea-back/src/database/migrations/1772400000000-CreateSuppliesAndPriceHistory.ts
  modified:
    - nemea-back/src/suppliers/suppliers.service.ts
    - nemea-back/src/suppliers/suppliers.module.ts
    - nemea-back/src/app.module.ts

key-decisions:
  - "Supply entity does NOT use @Index decorator for composite partial unique index -- defined in migration SQL to avoid TypeORM FK column name resolution issues"
  - "2-query pattern (find supplies + DISTINCT ON latest prices) instead of complex QueryBuilder subquery join for readability"
  - "Supplier cascade deactivation via direct Supply repository injection in SuppliersModule (avoids circular module dependency)"
  - "SupplyPriceHistory.price typed as string (TypeORM returns decimal as string) -- frontend parses with parseFloat"
  - "Supplier NOT editable after supply creation -- supplierId excluded from UpdateSupplyDto"

patterns-established:
  - "Append-only price history: SupplyPriceHistory records are immutable, no update/delete endpoints"
  - "Current price = most recent SupplyPriceHistory record, fetched via DISTINCT ON (supply_id) ORDER BY created_at DESC"
  - "Transactional create: entityManager.transaction() for atomic supply + initial price creation"
  - "Cascade deactivation: SuppliersService.toggleStatus deactivates supplies when deactivating supplier, no reactivation cascade"

requirements-completed: [SPPL-01, SPPL-02, SPPL-03, SPPL-04, SPPL-05]

# Metrics
duration: 4min
completed: 2026-03-05
---

# Phase 4 Plan 1: Supplies and Price History API Summary

**SuppliesModule with 7 REST endpoints, append-only price history via DISTINCT ON pattern, transactional create with optional initial price, and supplier cascade deactivation**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-05T21:11:17Z
- **Completed:** 2026-03-05T21:15:07Z
- **Tasks:** 2
- **Files modified:** 12

## Accomplishments
- Created Supply and SupplyPriceHistory entities with raw SQL migration (enum type, 2 tables, partial unique index, composite index)
- Built SuppliesModule with 7 REST endpoints: list/get/create/update/toggle-status supplies + get/add price history
- Implemented 2-query DISTINCT ON pattern for efficient current price retrieval (avoids N+1)
- Added supplier cascade deactivation directly in SuppliersService via Supply repository injection
- All 31 existing tests still pass, zero TypeScript errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Supply and SupplyPriceHistory entities + migration + UnitType enum** - `97de756` (feat)
2. **Task 2: SuppliesModule with CRUD + price history endpoints + supplier cascade** - `2e4912c` (feat)

## Files Created/Modified
- `nemea-back/src/supplies/entities/supply.entity.ts` - Supply entity with ManyToOne to SupplyType and Supplier, UnitType enum, is_active
- `nemea-back/src/supplies/entities/supply-price-history.entity.ts` - Append-only price history entity with decimal(12,2) price
- `nemea-back/src/supplies/dto/create-supply.dto.ts` - CreateSupplyDto with optional initialPrice
- `nemea-back/src/supplies/dto/update-supply.dto.ts` - UpdateSupplyDto (no supplierId, no initialPrice via OmitType)
- `nemea-back/src/supplies/dto/create-supply-price.dto.ts` - CreateSupplyPriceDto with price validation
- `nemea-back/src/supplies/supplies.service.ts` - Full CRUD + toggle-status + price history + deactivateBySupplier + 2-query current price
- `nemea-back/src/supplies/supplies.controller.ts` - 7 endpoints with Swagger docs and role guards
- `nemea-back/src/supplies/supplies.module.ts` - Imports Supply, SupplyPriceHistory, SupplyType, Supplier repos
- `nemea-back/src/database/migrations/1772400000000-CreateSuppliesAndPriceHistory.ts` - Raw SQL migration for supplies, supply_price_history, enum, indexes
- `nemea-back/src/suppliers/suppliers.service.ts` - Added cascade deactivation in toggleStatus + Supply repo injection
- `nemea-back/src/suppliers/suppliers.module.ts` - Added TypeOrmModule.forFeature([Supply]) import
- `nemea-back/src/app.module.ts` - Registered SuppliesModule

## Decisions Made
- Used raw SQL migration (not auto-generated) to explicitly create enum type and partial unique index, avoiding TypeORM decorator quirks with composite FK-based indexes
- Chose 2-query pattern (find + DISTINCT ON) over complex QueryBuilder subquery join -- cleaner code, same 2 SQL queries, easier to debug
- Injected Supply repository directly in SuppliersModule via TypeOrmModule.forFeature instead of importing SuppliesModule -- avoids circular module dependency
- SupplyPriceHistory.price typed as `string` (TypeORM decimal behavior) -- frontend will parseFloat for display
- UpdateSupplyDto uses OmitType to exclude supplierId and initialPrice from CreateSupplyDto -- enforces locked decision that supplier is not editable

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All 7 supply endpoints ready for frontend consumption in Plan 04-02
- SuppliesService exported for future Phase 6 cost calculation module
- Supplier cascade deactivation tested via SuppliersService.toggleStatus
- Migration will auto-run on next server boot (migrationsRun: true)

## Self-Check: PASSED

- FOUND: All 9 created files verified on disk
- FOUND: commit 97de756 (Task 1 entities + migration)
- FOUND: commit 2e4912c (Task 2 SuppliesModule + cascade)
- TypeScript: 0 errors
- Tests: 31/31 passing

---
*Phase: 04-supplies-and-price-history*
*Completed: 2026-03-05*
