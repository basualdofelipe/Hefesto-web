---
phase: 12-scenarios
verified: 2026-03-29T05:30:00Z
status: passed
score: 5/5 must-haves verified
re_verification: false
gaps: []
human_verification:
  - test: "Create a named scenario, set override prices for 2 products, click Guardar y calcular"
    expected: "Margin summary card appears showing real vs simulated margins with color-coded differences; overrides persist on page reload"
    why_human: "Requires live browser session with authenticated user and seeded DB data"
  - test: "Create scenario for user A; log in as user B and attempt GET /api/scenarios/:id for user A's private scenario"
    expected: "404 returned — user B cannot see user A's private scenario"
    why_human: "Requires two authenticated sessions against a running backend"
  - test: "Apply bulk +10% adjustment to Cinturones type, then click Guardar y calcular"
    expected: "All Cinturon products show updated override prices; margin columns reflect new values"
    why_human: "Requires live browser interaction with product data loaded"
---

# Phase 12: Scenarios Verification Report

**Phase Goal:** Investors can create what-if scenarios with overridden selling prices and see recalculated margins without touching real data
**Verified:** 2026-03-29T05:30:00Z
**Status:** passed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can create a named scenario that persists in DB | VERIFIED | `ScenariosService.create()` saves to `scenarios` table via TypeORM; migration creates table; ConflictException on duplicate name per user |
| 2 | User can set override selling prices for specific products within a scenario | VERIFIED | `upsertOverrides()` wraps delete+insert in a QueryRunner transaction; `ScenarioEditorClient` sends PUT `/api/scenarios/:id/overrides`; `ProductOverrideTable` renders debounced inputs with blue highlight |
| 3 | User can apply bulk price adjustments and see recalculated margins | VERIFIED | `BulkOverrideDialog` applies percentage to product type or all; `handleBulkOverride` in `ScenarioEditorClient` updates overrides state; `handleSaveAndCalculate` POSTs to calculate endpoint |
| 4 | Scenario shows recalculated margins using real costs and overridden prices | VERIFIED | `calculate()` loads all products, costs via `costsService.calculateAll()`, and calls `calculadoraService.calcForward()` per product; both simResult (override price) and realResult (real price) returned; `ProductOverrideTable` renders both with color-coded diff |
| 5 | Scenarios are user-scoped and persist in DB | VERIFIED | `findAll()` uses two queries (own + public from others via `Not(userId)`); `findOne` enforces owner-or-public check; other users get 404; no writes to `product_price_history` table |

**Score:** 5/5 truths verified

### Required Artifacts

#### Plan 01 (Backend)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/scenarios/entities/scenario.entity.ts` | Scenario entity with user FK, gateway_slug, plan FK, is_public | VERIFIED | `@Entity('scenarios')`, user ManyToOne, plan ManyToOne nullable, isPublic column, all gateway fields present |
| `nemea-back/src/scenarios/entities/scenario-override.entity.ts` | ScenarioOverride with scenario FK, product FK, override_price | VERIFIED | `@Entity('scenario_overrides')`, `@Unique(['scenario','product'])`, overridePrice decimal(12,2) |
| `nemea-back/src/scenarios/scenarios.service.ts` | CRUD + calculate with user scoping | VERIFIED | 7 methods: create, findAll, findOne, update, remove, togglePublic, upsertOverrides, calculate — all owner-checked |
| `nemea-back/src/scenarios/scenarios.controller.ts` | 8 REST endpoints | VERIFIED | POST /, GET /, GET /:id, PUT /:id, DELETE /:id, PUT /:id/overrides, GET /:id/calculate, PATCH /:id/toggle-public |
| `nemea-back/src/scenarios/scenarios.module.ts` | Module importing CalculadoraModule, CostsModule, ProductsModule, TiendanubeConfigModule | VERIFIED | All 4 modules imported; ScenariosService and ScenariosController registered |
| `nemea-back/src/database/migrations/1772900000000-CreateScenariosTables.ts` | Migration creating scenarios + scenario_overrides tables | VERIFIED | Creates both tables with FKs, unique constraint, indexes on user_id and scenario_id |
| `nemea-back/src/scenarios/scenarios.service.spec.ts` | Unit tests for create, calculate, user scoping | VERIFIED | 7 tests all passing (confirmed by test run) |

#### Plan 02 (Frontend)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-front/src/components/scenarios/types.ts` | Scenario, ScenarioOverride, ScenarioProductResult, ScenarioCalcResponse types | VERIFIED | All 7 interfaces present: ScenarioUser, ScenarioOverride, Scenario, ScenarioProductResult, ScenarioCalcResult, ScenarioCalcResponse, GatewayPlanConfig + CreateScenarioPayload + OverrideItem |
| `nemea-front/src/app/(app)/escenarios/page.tsx` | RSC page that fetches scenarios list | VERIFIED | `apiFetch` to `/api/scenarios` and `/api/tiendanube-config/all` in Promise.all; passes to `ScenarioListClient` |
| `nemea-front/src/app/(app)/escenarios/ScenarioListClient.tsx` | Client component managing scenario list state and CRUD | VERIFIED | useState for scenarios, handles delete/togglePublic/create; separates own vs shared scenarios |
| `nemea-front/src/app/(app)/escenarios/[id]/page.tsx` | RSC page that fetches scenario detail + products + config | VERIFIED | Parallel `apiFetch` for scenario, products, config |
| `nemea-front/src/app/(app)/escenarios/[id]/ScenarioEditorClient.tsx` | Client component with full scenario editor | VERIFIED | overridesState, gatewayPlanConfig, calcResults state; handleSaveAndCalculate with parallel PUT then sequential GET |
| `nemea-front/src/components/scenarios/ProductOverrideTable.tsx` | Editable product table with override prices and margin columns | VERIFIED | DebouncedInput (300ms, key-based remount), blue highlight for overrides, simResult/realResult columns, color-coded diff |
| `nemea-front/src/components/scenarios/BulkOverrideDialog.tsx` | Dialog for bulk percentage price adjustment | VERIFIED | product type select, percentage input, preview count, applies on top of existing overrides |
| `nemea-front/src/components/layout/AppSidebar.tsx` | Escenarios link added to HERRAMIENTAS_ITEMS | VERIFIED | Line 45: `{ label: 'Escenarios', href: '/escenarios', icon: LineChart }` in HERRAMIENTAS_ITEMS |

### Key Link Verification

#### Plan 01 (Backend)

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| scenarios.service.ts | calculadora.service.ts | `this.calculadoraService.calcForward()` | WIRED | Lines 312, 330 — called for both simResult and realResult per product |
| scenarios.service.ts | costs.service.ts | `this.costsService.calculateAll()` | WIRED | Line 259 |
| scenarios.service.ts | products.service.ts | `this.productsService.findAll(true)` | WIRED | Line 262 — includes inactive products |
| scenarios.service.ts | tiendanube-config.service.ts | `this.tiendanubeConfigService.getAll()` | WIRED | Line 256 |
| app.module.ts | scenarios.module.ts | `ScenariosModule` import | WIRED | Lines 17 (import) and 42 (AppModule imports array) |

#### Plan 02 (Frontend)

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| escenarios/page.tsx | /api/scenarios | `apiFetch` | WIRED | Line 9 — fetches list in parallel with config |
| ScenarioEditorClient.tsx | /api/scenarios/:id/overrides | `apiClientFetch PUT` | WIRED | Line 131 |
| ScenarioEditorClient.tsx | /api/scenarios/:id/calculate | `apiClientFetch GET` | WIRED | Line 151 — sequential after parallel saves |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| ProductOverrideTable.tsx | `calcResults` (simResult/realResult columns) | ScenarioEditorClient → GET /api/scenarios/:id/calculate → ScenariosService.calculate() → costsService.calculateAll() + productsService.findAll() + calcForward() | Yes — real DB queries for costs and products, real computation formula | FLOWING |
| ScenarioListClient.tsx | `scenarios` (initial list) | page.tsx → apiFetch → GET /api/scenarios → ScenariosService.findAll() → scenarioRepo.find() | Yes — TypeORM queries against `scenarios` table | FLOWING |
| MarginSummary.tsx | `results` (avgMargin, totalMargin, overrideCount) | Passed from ScenarioEditorClient.calcResults which comes from calculate endpoint | Yes — flows from real calculate result | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| ScenariosService unit tests pass | `npx jest scenarios.service.spec.ts --no-coverage` | 7 tests passed in 1.677s | PASS |
| ScenariosModule registered in AppModule | `grep ScenariosModule nemea-back/src/app.module.ts` | Lines 17 (import) and 42 (array) found | PASS |
| Migration file exists with CREATE TABLE | `ls migrations/ | grep Scenario` | `1772900000000-CreateScenariosTables.ts` found | PASS |
| Sidebar link present | `grep -n escenarios AppSidebar.tsx` | Line 45 in HERRAMIENTAS_ITEMS | PASS |
| product_price_history never touched | `grep -r product_price_history src/scenarios/` | No matches | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| SCEN-01 | 12-01, 12-02 | User can create a named scenario with overridden selling prices for specific products | SATISFIED | `ScenariosService.create()` + `upsertOverrides()` in backend; `CreateScenarioDialog` + `ProductOverrideTable` in frontend |
| SCEN-02 | 12-01, 12-02 | User can apply bulk price adjustments (e.g., "+10% all cinturones") | SATISFIED | `BulkOverrideDialog` + `handleBulkOverride()` in ScenarioEditorClient applies percentage by type or all |
| SCEN-03 | 12-01, 12-02 | Scenario shows recalculated margins for all affected products using real costs + overridden prices | SATISFIED | `calculate()` endpoint calls `calcForward()` per product with override-first effective price; `ProductOverrideTable` shows real vs sim columns with color diff |
| SCEN-04 | 12-01, 12-02 | Scenarios are user-scoped and persist in DB | SATISFIED | `findAll()` returns own + public from others; `findOne` throws 404 for non-owner non-public; `scenarios` table with user_id FK; no scenario data written to product_price_history |

All 4 requirements satisfied. All are marked `[x]` in REQUIREMENTS.md Simulator section.

**No orphaned requirements detected.** REQUIREMENTS.md traceability table shows SCEN-01 through SCEN-04 all mapped to Phase 12 and marked Complete.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None | — | No stubs, TODO/FIXME, empty returns, or hollow props detected | — | — |

**Stub check summary:**
- All `placeholder` matches in frontend are HTML input placeholder attributes, not code stubs
- No `return null` in service methods (only in per-product error isolation which is intentional)
- No hardcoded empty arrays returned from API routes
- No `console.log`-only handlers

### Human Verification Required

#### 1. Create scenario + set overrides + calculate

**Test:** Log in as any user. Navigate to /escenarios. Create a scenario named "Test Enero 2027". In the editor, enter an override price for 2 products. Click "Guardar y calcular".
**Expected:** Margin summary card appears. ProductOverrideTable shows real margin column populated, simulated margin column shows different values where overrides were set, and the diff column shows green or red arrows. After page reload the override prices persist.
**Why human:** Requires authenticated browser session, live backend, and seeded product/config data.

#### 2. User isolation (SCEN-04 boundary)

**Test:** Create a private scenario as user A. Log in as user B. Attempt to navigate to /escenarios/[user-A-scenario-id].
**Expected:** The page shows a 404 or "not found" state — user B cannot view user A's private scenario.
**Why human:** Requires two distinct authenticated accounts to test cross-user boundary.

#### 3. Bulk override by product type

**Test:** In a scenario editor with at least 3 Cinturon products loaded, open "Ajuste masivo". Select the Cinturon type. Enter +15. Click "Aplicar ajuste".
**Expected:** All Cinturon rows in the ProductOverrideTable show blue-highlighted override inputs populated with original_price * 1.15 (rounded to integer). Non-Cinturon rows remain unchanged.
**Why human:** Requires live product data of multiple types to verify type filter works correctly.

### Gaps Summary

No gaps found. All 5 truths verified, all 15 artifacts exist and are substantive, all 8 key links are wired, data flows from DB through to UI, 7 unit tests pass, and no anti-patterns found. Phase 12 goal is fully achieved in code.

The only open items are behavioral spot-checks that require human verification with a live environment (authenticated sessions, running backend, seeded data). These are `human_needed` items, not gaps — the code is complete.

---

_Verified: 2026-03-29T05:30:00Z_
_Verifier: Claude (gsd-verifier)_
