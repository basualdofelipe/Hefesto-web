---
phase: 10-tiendanube-config
plan: 01
subsystem: backend
tags: [tiendanube, config, entities, migration, seed, rest-api]
dependency_graph:
  requires: []
  provides: [TiendanubeConfigModule, TiendanubeConfigService, 5 TN config entities, 9 REST endpoints]
  affects: [app.module.ts]
tech_stack:
  added: []
  patterns: [append-only-history, DISTINCT-ON-latest, decimal-parse-at-service-boundary]
key_files:
  created:
    - nemea-back/src/tiendanube-config/entities/tn-payment-gateway.entity.ts
    - nemea-back/src/tiendanube-config/entities/tn-gateway-rate.entity.ts
    - nemea-back/src/tiendanube-config/entities/tn-installment-rate.entity.ts
    - nemea-back/src/tiendanube-config/entities/tn-tax-config.entity.ts
    - nemea-back/src/tiendanube-config/entities/tn-plan.entity.ts
    - nemea-back/src/tiendanube-config/tiendanube-config.service.ts
    - nemea-back/src/tiendanube-config/tiendanube-config.controller.ts
    - nemea-back/src/tiendanube-config/tiendanube-config.module.ts
    - nemea-back/src/tiendanube-config/dto/update-gateway-rate.dto.ts
    - nemea-back/src/tiendanube-config/dto/update-installment-rate.dto.ts
    - nemea-back/src/tiendanube-config/dto/update-tax-config.dto.ts
    - nemea-back/src/tiendanube-config/dto/update-plan.dto.ts
    - nemea-back/src/database/migrations/1772800000000-CreateTiendanubeConfigTables.ts
  modified:
    - nemea-back/src/app.module.ts
decisions:
  - ParseIntPipe on installment-rates/:installments param (integer, not UUID)
  - Decimal columns parsed to number at service boundary with parseFloat
  - DISTINCT ON raw queries for latest rates (same pattern as supplies.service.ts)
  - row_to_json used in gateway rates query to include gateway relation data
metrics:
  duration: 4min
  completed: 2026-03-27
  tasks: 2/2
  files_created: 13
  files_modified: 1
---

# Phase 10 Plan 01: Tiendanube Config Backend Summary

TiendanubeConfigModule with 5 entities, append-only rate history, migration seeded with verified March 2026 gateway/installment/tax data, and 9 REST endpoints with admin-only mutations.

## What Was Built

### Task 1: 5 Entities + Migration with Seed Data (f88e930)

Created 5 TypeORM entities extending BaseEntity (UUID PK, timestamps):

1. **TnPaymentGateway** - Gateway master (slug, label, isActive). 3 gateways seeded: Pago Nube (active), Mercado Pago (active), MODO (inactive).
2. **TnGatewayRate** - Append-only rate history per gateway/method/withdrawal combo. 13 rates seeded across 3 gateways. Composite index on (gateway_id, payment_method, withdrawal_days, created_at DESC).
3. **TnInstallmentRate** - Append-only installment fee history. 5 tiers seeded (1/3/6/9/12 cuotas).
4. **TnTaxConfig** - Append-only tax config (IVA + IIBB). Seeded with 21% IVA, 3.5% IIBB.
5. **TnPlan** - Tiendanube subscription plans with CPT rates. 4 plans seeded (Inicial, Esencial, Impulso, Escala).

Migration creates tables in dependency order, seeds with exact March 2026 rates from CONTEXT.md, and uses subqueries to reference gateway IDs.

### Task 2: Module, Service, Controller, DTOs (6d802ce)

**Service** (TiendanubeConfigService):
- `getAll()` - Returns combined config with latest rates via DISTINCT ON queries
- `getGatewaysWithRates()` - Gateways with their latest rates grouped
- `updateGatewayRate()` - Append-only: creates new rate record
- `getInstallmentRates()` - Latest rate per installment count via DISTINCT ON
- `updateInstallmentRate()` - Append-only: creates new installment rate
- `getTaxConfig()` - Latest tax config row
- `updateTaxConfig()` - Append-only: creates new tax config row
- `getPlans()` - All active plans
- `updatePlan()` - Direct update on plan CPT rates (not append-only)

All decimal columns (ratePercent, ivaRate, iibbRate, cptPagoNube, cptOtherGateways) parsed from string to number at service boundary using parseFloat. Frontend always receives numbers.

**Controller** (9 endpoints):
- GET /tiendanube-config/all - Any authenticated user
- GET /tiendanube-config/gateways - Any authenticated user
- PUT /tiendanube-config/gateway-rates/:gatewayId - ADMIN only (ParseUUIDPipe)
- GET /tiendanube-config/installments - Any authenticated user
- PUT /tiendanube-config/installment-rates/:installments - ADMIN only (ParseIntPipe)
- GET /tiendanube-config/taxes - Any authenticated user
- PUT /tiendanube-config/taxes - ADMIN only
- GET /tiendanube-config/plans - Any authenticated user
- PUT /tiendanube-config/plans/:id - ADMIN only (ParseUUIDPipe)

**DTOs** with class-validator: UpdateGatewayRateDto, UpdateInstallmentRateDto, UpdateTaxConfigDto, UpdatePlanDto.

**Module** registered in AppModule after ExpensesModule. Exports TiendanubeConfigService for Phase 11 CalculadoraModule.

## Deviations from Plan

None - plan executed exactly as written.

## Verification Results

1. `npx tsc --noEmit` - PASSED (zero errors)
2. `npm run lint` - PASSED (zero errors)
3. TiendanubeConfigModule in AppModule - CONFIRMED (2 occurrences: import + registration)
4. Migration contains 5 CREATE TABLE + 17 INSERT statements - CONFIRMED
5. 5 entity files with correct TypeORM decorators - CONFIRMED
6. All PUT endpoints have @Roles(Role.ADMIN) - CONFIRMED (4 PUT endpoints)
7. GET endpoints have no @Roles - CONFIRMED (5 GET endpoints, all public to authenticated users)
8. Service methods parse decimal strings to numbers - CONFIRMED (6 parseFloat calls)

## Known Stubs

None - all data is sourced from database via seed migration.

## Self-Check: PASSED
