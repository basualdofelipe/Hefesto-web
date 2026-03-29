---
phase: 12-scenarios
plan: 01
subsystem: backend
tags: [scenarios, crud, calculate, user-scoping, typeorm]
dependency_graph:
  requires: [CalculadoraModule, CostsModule, ProductsModule, TiendanubeConfigModule]
  provides: [ScenariosModule, ScenariosService, ScenariosController]
  affects: [AppModule]
tech_stack:
  added: []
  patterns: [QueryRunner transaction, user-scoped visibility, per-product error isolation]
key_files:
  created:
    - nemea-back/src/scenarios/entities/scenario.entity.ts
    - nemea-back/src/scenarios/entities/scenario-override.entity.ts
    - nemea-back/src/scenarios/dto/create-scenario.dto.ts
    - nemea-back/src/scenarios/dto/update-scenario.dto.ts
    - nemea-back/src/scenarios/dto/upsert-overrides.dto.ts
    - nemea-back/src/scenarios/dto/scenario-response.dto.ts
    - nemea-back/src/scenarios/scenarios.service.ts
    - nemea-back/src/scenarios/scenarios.controller.ts
    - nemea-back/src/scenarios/scenarios.module.ts
    - nemea-back/src/scenarios/scenarios.service.spec.ts
    - nemea-back/src/database/migrations/1772900000000-CreateScenariosTables.ts
  modified:
    - nemea-back/src/app.module.ts
decisions:
  - "Scenario overrides use full-replace strategy (delete+insert in transaction) not incremental upsert"
  - "calculate returns both simResult (override/effective price) and realResult (current price) per product"
  - "findAll uses two separate queries merged (own + public) instead of QueryBuilder"
  - "Test mock fix: 4 calcForward mock calls needed (sim+real per product) not 2"
metrics:
  duration: 8min
  completed: "2026-03-29T04:08Z"
  tasks: 3
  files: 12
requirements: [SCEN-01, SCEN-02, SCEN-03, SCEN-04]
---

# Phase 12 Plan 01: ScenariosModule Backend Summary

ScenariosModule with CRUD, user-scoped visibility, transactional override upsert, and calculate endpoint delegating to CalculadoraService.calcForward() with per-product error isolation.

## What Was Built

### Entities (2 new DB tables)
- **Scenario** (`scenarios` table): UUID PK, name, user FK, isPublic flag, gateway config fields (gatewaySlug, paymentMethod, withdrawalDays, installments), plan FK to tn_plans
- **ScenarioOverride** (`scenario_overrides` table): UUID PK, scenario FK (CASCADE delete), product FK, overridePrice (decimal 12,2), composite unique on (scenario, product)

### Migration
- `1772900000000-CreateScenariosTables.ts`: Creates both tables with FKs, indexes on user_id and scenario_id, unique constraint on (scenario_id, product_id)

### DTOs (4 files)
- `CreateScenarioDto`: class-validator with @IsNotEmpty, @MaxLength(200), optional gateway/plan fields
- `UpdateScenarioDto`: PartialType of create
- `UpsertOverridesDto`: Array of @ValidateNested OverrideItemDto with @IsUUID productId and @Min(0) overridePrice
- `ScenarioCalcResponse` / `ScenarioProductResult`: Typed interfaces for calculate endpoint response

### ScenariosService (7 methods)
1. **create**: Name uniqueness per user (ConflictException), creates with user FK
2. **findAll**: Two queries merged (own + public from others), relations: user+plan only (no eager override loading)
3. **findOne**: Owner or public access with full override relations
4. **update**: Owner check + name uniqueness on change
5. **remove**: Owner check + cascade delete
6. **togglePublic**: Owner-only toggle
7. **upsertOverrides**: QueryRunner transaction wrapping delete+insert (review fix)
8. **calculate**: Loads all products including inactive via findAll(true), builds override map, calls calcForward per product with try/catch error isolation

### ScenariosController (8 REST endpoints)
- POST / -- create scenario
- GET / -- list own + public
- GET /:id -- get with overrides
- PUT /:id -- update metadata
- DELETE /:id -- delete with cascade
- PUT /:id/overrides -- bulk upsert overrides
- GET /:id/calculate -- calculate margins
- PATCH /:id/toggle-public -- toggle visibility

All endpoints use @Roles(Role.ADMIN, Role.USER), @CurrentUser('id'), ParseUUIDPipe.

### Unit Tests (7 passing)
- Service defined
- Create with valid data + ConflictException on duplicate name
- findAll returns own + public from others
- Update throws NotFoundException for non-owner
- Calculate delegates to calcForward with effective price (override > real)
- Calculate handles per-product calcForward errors without crashing loop

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Test mock sequence for per-product error handling**
- **Found during:** Task 2 verification
- **Issue:** Test expected 2 calcForward mock calls but calculate() makes 4 calls (simResult + realResult per product). Mock sequence was consumed incorrectly.
- **Fix:** Updated test to mock 4 calls (sim+real for prod-1 both throw, sim+real for prod-2 both succeed). Added assertion for realResult null on error product.
- **Files modified:** nemea-back/src/scenarios/scenarios.service.spec.ts
- **Commit:** b61436e

## Verification Results

| Check | Result |
|-------|--------|
| TypeScript compilation (tsc --noEmit) | PASS -- 0 errors |
| Unit tests (7/7) | PASS |
| Endpoint count | 8 (matches spec) |
| ScenariosModule in AppModule | Registered |
| Migration file exists | Yes |
| No ProductPriceHistory in scenarios/ | CLEAN (data isolation enforced) |
| findAll(true) in service | Present (includes inactive products) |
| startTransaction in service | Present (transactional upsert) |
| Per-product catch in calculate | Present (error isolation) |

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 0 | 1e76749 | test(12-01): add failing test scaffold for ScenariosService |
| 1 | ef9656f | feat(12-01): add Scenario entities, DTOs, migration, module shell |
| 2 | b61436e | feat(12-01): implement ScenariosService CRUD + calculate + controller |

## Known Stubs

None -- all endpoints are fully implemented with real service logic. No placeholder text or hardcoded empty values.

## Self-Check: PASSED

All 11 created files verified present. All 3 commit hashes verified in git log.
