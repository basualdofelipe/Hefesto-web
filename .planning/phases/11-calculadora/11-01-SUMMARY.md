---
phase: 11-calculadora
plan: 01
subsystem: backend
tags: [calculadora, pricing, tiendanube, tdd, forward, inverse, batch]
dependency_graph:
  requires: [TiendanubeConfigModule, CostsModule, ProductsModule]
  provides: [CalculadoraModule, CalculadoraService, 3 POST endpoints]
  affects: [app.module.ts]
tech_stack:
  added: []
  patterns: [binary-search-inverse, snake-case-raw-sql-workaround, percentage-to-fraction-conversion]
key_files:
  created:
    - nemea-back/src/calculadora/calculadora.service.ts
    - nemea-back/src/calculadora/calculadora.controller.ts
    - nemea-back/src/calculadora/calculadora.module.ts
    - nemea-back/src/calculadora/dto/calc-forward.dto.ts
    - nemea-back/src/calculadora/dto/calc-inverse.dto.ts
    - nemea-back/src/calculadora/dto/calc-batch.dto.ts
    - nemea-back/src/calculadora/dto/calc-result.dto.ts
    - nemea-back/src/calculadora/calculadora.service.spec.ts
  modified:
    - nemea-back/src/app.module.ts
decisions:
  - Cast through unknown for snake_case field access on raw SQL results (TypeScript strict compliance)
  - IVA/IIBB converted in resolveRates (percentage to fraction), gateway/installment/CPT stay as percentages (divided by 100 inline in formula)
  - calcBatch passes costoEnvio=0 (batch mode has no per-product shipping)
  - Upper bound for calcInverse binary search: max(costoProducto * 20, 100000)
metrics:
  duration: 9min
  completed: 2026-03-27
  tasks: 2/2
  files_created: 8
  files_modified: 1
---

# Phase 11 Plan 01: Calculadora Backend (TDD) Summary

CalculadoraService with 14-step forward pricing formula, binary search inverse, batch margin calculation, and 6 TDD unit tests including Hefesto $87,000 round-trip verification.

## What Was Built

### Task 1: DTOs + CalculadoraService with TDD Tests (02bb344 RED, af0ecc6 GREEN)

**TDD RED phase** -- wrote 6 failing tests with realistic mock data reproducing real runtime bugs:
- Mock gateway rates use snake_case field names (`payment_method`, `withdrawal_days`, `rate_percent`) matching raw SQL output
- Mock `ratePercent: NaN` on every rate object (reproducing the parseFloat(undefined) bug)
- Mock `ivaRate: 21` and `iibbRate: 3.5` as percentages (not fractions)
- Mock `currentPrice` as strings (`"87000.00"`) matching TypeORM decimal column behavior

**TDD GREEN phase** -- implemented CalculadoraService:

1. **resolveRates()** -- Private helper that finds matching gateway rate, installment rate, tax config, and CPT from TiendanubeConfigAll. Handles 3 critical bugs:
   - Reads `rate_percent` via bracket notation (not broken `ratePercent` which is NaN)
   - Reads `payment_method` and `withdrawal_days` via bracket notation (not undefined camelCase)
   - Converts IVA/IIBB from percentages to fractions (21 -> 0.21, 3.5 -> 0.035)
   - Checks taxConfig for null with clear error message

2. **calcForward()** -- Pure math, 14-step pricing formula adapted from prototype:
   - totalCliente, tasaBase, tasaConIVA, comisionPasarela
   - tasaCuotas, costoFinanciacion
   - CPT (Tiendanube transaction cost -- new vs prototype)
   - baseGravada, ivaDebito, ivaCreditoProducto, ivaCreditoComision, ivaNeto
   - retencionIIBB, netoRecibido (subtracts CPT)
   - costoProductoConIVA, gananciaReal, margen
   - Rounds only gananciaReal and margen at the end (no intermediate rounding)

3. **calcInverse()** -- Binary search over calcForward:
   - Epsilon convergence: `while (high - low > 0.01 && iterations < 100)`
   - Edge case validation: zero cost, negative profit, unreachable target
   - Upper bound: `Math.max(costoProducto * 20, 100000)`
   - Returns CalcError object for edge cases, CalcInverseResult for success

4. **calcBatch()** -- Batch margin calculation for all products:
   - Loads TN config once, costs once, products once (~5 SQL queries total)
   - Runs calcForward in-memory for each product with a valid price
   - `parseFloat(product.currentPrice)` handles TypeORM decimal string conversion
   - Builds display name from type + name + finish

**DTOs created:**
- `CalcForwardDto` -- productId (optional UUID), precioVenta, costoEnvio, costoProducto (optional), gateway params
- `CalcInverseDto` -- same but gananciaDeseada instead of precioVenta
- `CalcBatchDto` -- gateway params only (applies to all products)
- `CalcResult`, `CalcInverseResult`, `CalcBatchItem`, `CalcError` interfaces

**6 unit tests:**
1. calcForward Hefesto at $87,000 -- verifies totalCliente=94315, positive gananciaReal, all 16 fields
2. calcInverse round-trip -- forward($87,000) -> gananciaReal -> inverse -> precioVenta within $0.01
3. calcInverse zero cost -> "Defini el costo del producto primero"
4. calcInverse negative profit -> "La ganancia deseada debe ser positiva"
5. calcInverse unreachable target -> "Ganancia inalcanzable con estas tasas"
6. calcBatch parses string currentPrice, returns 3 products with numeric prices and non-null results

### Task 2: CalculadoraModule + Controller + AppModule Registration (2292a4b)

**CalculadoraController** with 3 POST endpoints:
- `POST /calculadora/forward` -- accepts CalcForwardDto, returns CalcResult directly
- `POST /calculadora/inverse` -- accepts CalcInverseDto, throws BadRequestException on CalcError
- `POST /calculadora/batch` -- accepts CalcBatchDto, returns CalcBatchItem[]

All endpoints have `@Roles(Role.ADMIN, Role.USER)` -- both roles can access the calculator.
All have Swagger decorators (`@ApiBearerAuth`, `@ApiOperation`, `@ApiResponse`, `@ApiTags`).
Controller returns values directly -- ResponseInterceptor wraps to `{ data: T }`.

Forward and inverse endpoints support `productId` -- if provided, cost is fetched from `CostsService.calculateForProduct()`.

**CalculadoraModule** imports TiendanubeConfigModule, CostsModule, ProductsModule. Exports CalculadoraService for Phase 12 (Scenarios) and Phase 13 (Dashboard).

Registered in AppModule after TiendanubeConfigModule.

## Deviations from Plan

None -- plan executed exactly as written.

## Verification Results

1. `npx jest calculadora --verbose` -- PASSED (6/6 tests)
2. `npx tsc --noEmit` -- PASSED (zero type errors)
3. `npm run lint` -- PASSED (zero lint errors)
4. `grep CalculadoraModule app.module.ts` -- CONFIRMED (import + registration)
5. `grep calcForward/calcInverse/calcBatch calculadora.service.ts` -- CONFIRMED (all 3 methods)
6. `grep 87000 calculadora.service.spec.ts` -- CONFIRMED (Hefesto test case: 7 occurrences)
7. `grep parseFloat calculadora.service.ts` -- CONFIRMED (3 locations: resolveRates x2, calcBatch x1)
8. `grep payment_method/withdrawal_days calculadora.service.ts` -- CONFIRMED (snake_case access)
9. `grep rate_percent calculadora.service.ts` -- CONFIRMED (bracket notation access: 2 locations)
10. `grep "/ 100" calculadora.service.ts` -- CONFIRMED (IVA/IIBB conversion + inline formula divisions)

## Known Stubs

None -- all methods fully implemented with real logic.

## Self-Check: PASSED

All 8 created files verified on disk. All 3 commit hashes verified in git log.
