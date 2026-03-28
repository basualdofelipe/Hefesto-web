---
phase: 10-tiendanube-config
plan: 03
subsystem: backend
tags: [bugfix, raw-sql, camelcase, tiendanube-config, calculadora]
dependency_graph:
  requires: [10-01, 10-02]
  provides: [camelCase-api-response]
  affects: [tiendanube-config-page, calculadora-page, gateway-selectors]
tech_stack:
  patterns: [raw-sql-column-aliases, json_build_object]
key_files:
  modified:
    - nemea-back/src/tiendanube-config/tiendanube-config.service.ts
    - nemea-back/src/calculadora/calculadora.service.ts
    - nemea-back/src/calculadora/calculadora.service.spec.ts
decisions:
  - "Use json_build_object instead of row_to_json for explicit camelCase key control in nested gateway objects"
  - "Remove isNaN safety checks in CalculadoraService since ratePercent is now always a valid number from parseGatewayRate"
metrics:
  duration: 3m 22s
  completed: 2026-03-28T20:37:03Z
  tasks: 2/2
  files_modified: 3
---

# Phase 10 Plan 03: Raw SQL snake_case Column Alias Fix Summary

Fixed raw SQL queries returning snake_case field names that caused NaN/null values for all rate fields, crashing the /configuracion/tiendanube page and breaking GatewaySelectors on /calculadora.

## What Changed

### Task 1: camelCase column aliases in TiendanubeConfigService (3a5ee94)

- **getLatestGatewayRates()**: Replaced `gr.*` with explicit columns using `AS "camelCase"` aliases (paymentMethod, withdrawalDays, ratePercent, isActive, createdAt, updatedAt, gatewayId). Replaced `row_to_json(gw.*)` with `json_build_object(...)` to produce camelCase keys in nested gateway JSON.
- **getInstallmentRates()**: Replaced `SELECT DISTINCT ON (installments) *` with explicit columns and camelCase aliases.
- **getGatewaysWithRates()**: Removed `(r as { gateway_id?: string }).gateway_id` snake_case fallback since `r.gateway.id` now always works.

### Task 2: Clean CalculadoraService and test mocks (74f15a2)

- **resolveRates()**: Removed all `as unknown as Record<string, unknown>` casts and bracket notation (`raw['payment_method']`, `raw['rate_percent']`, etc.). Now uses clean `rate.paymentMethod`, `rate.withdrawalDays`, `matchedRate.ratePercent` directly.
- **Removed CRITICAL BUG comments**: The bug is fixed at source (SQL aliases), workaround comments no longer relevant.
- **Test mocks**: Updated from broken snake_case shape (`payment_method`, `rate_percent: '3.49'`, `ratePercent: NaN`) to fixed camelCase shape (`paymentMethod`, `ratePercent: 3.49` as number).

## Root Cause

The two raw SQL methods (`getLatestGatewayRates`, `getInstallmentRates`) returned PostgreSQL column names in snake_case (`rate_percent`, `payment_method`, `withdrawal_days`), but all parse helpers (`parseGatewayRate`, `parseInstallmentRate`) and downstream consumers expected camelCase entity field names (`ratePercent`, `paymentMethod`, `withdrawalDays`). This caused `parseFloat(undefined)` to return `NaN` for every rate.

## Verification Results

- TypeScript compilation: zero errors
- ESLint: zero errors
- Calculadora tests: 6/6 passed (including Hefesto $87,000 round-trip)
- Grep for snake_case in calculadora/: zero hits
- Grep for camelCase aliases in SQL: confirmed present

## Commits

| Task | Hash | Message |
|------|------|---------|
| 1 | 3a5ee94 | fix(10-03): add camelCase column aliases to raw SQL queries |
| 2 | 74f15a2 | fix(10-03): remove snake_case workarounds from CalculadoraService |

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all data flows are fully wired.

## UAT Gaps Addressed

- **Test 1 (blocker)**: Config page crashes with TypeError on `null.toFixed()` -- FIXED. `ratePercent` is now a valid number, not null/NaN.
- **Tests 2-6 (blocked by Test 1)**: All unblocked by this fix. Page will load, enabling rate editing, tax editing, plan selection, and link verification.

## Self-Check: PASSED

- 10-03-SUMMARY.md: FOUND
- Commit 3a5ee94: FOUND
- Commit 74f15a2: FOUND
- tiendanube-config.service.ts: FOUND
- calculadora.service.ts: FOUND
- calculadora.service.spec.ts: FOUND
