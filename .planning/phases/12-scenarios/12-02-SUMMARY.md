---
phase: 12-scenarios
plan: 02
subsystem: frontend
tags: [scenarios, what-if, override-prices, margin-comparison, shadcn, debounced-input, react]
dependency_graph:
  requires:
    - phase: 12-01
      provides: ScenariosModule REST API (8 endpoints)
    - phase: 11-02
      provides: CalculadoraClient patterns, GatewaySelectors reference
    - phase: 10-02
      provides: TiendanubeConfigAll type, PAYMENT_METHOD_LABELS
    - phase: 06-02
      provides: Product types, formatCost, formatSellingPrice, getProductDisplayName
  provides:
    - /escenarios list page with create/delete/toggle-public
    - /escenarios/[id] editor with product override table
    - Scenario types (Scenario, ScenarioOverride, ScenarioCalcResponse, GatewayPlanConfig)
    - 7 scenario components (ScenarioCard, CreateScenarioDialog, ProductOverrideTable, BulkOverrideDialog, GatewayPlanSelector, MarginSummary, types)
    - Sidebar integration under Herramientas for all users
  affects: [investor-dashboard, AppSidebar]
tech_stack:
  added: []
  patterns: [DebouncedInput with defaultValue+key remount, ref-based callback for cascading selects, parallel API saves with Promise.all, override-first base price for bulk adjustments]
key_files:
  created:
    - nemea-front/src/components/scenarios/types.ts
    - nemea-front/src/components/scenarios/ScenarioCard.tsx
    - nemea-front/src/components/scenarios/CreateScenarioDialog.tsx
    - nemea-front/src/components/scenarios/ProductOverrideTable.tsx
    - nemea-front/src/components/scenarios/BulkOverrideDialog.tsx
    - nemea-front/src/components/scenarios/GatewayPlanSelector.tsx
    - nemea-front/src/components/scenarios/MarginSummary.tsx
    - nemea-front/src/app/(app)/escenarios/page.tsx
    - nemea-front/src/app/(app)/escenarios/ScenarioListClient.tsx
    - nemea-front/src/app/(app)/escenarios/[id]/page.tsx
    - nemea-front/src/app/(app)/escenarios/[id]/ScenarioEditorClient.tsx
  modified:
    - nemea-front/src/components/layout/AppSidebar.tsx
decisions:
  - "DebouncedInput uses uncontrolled input with defaultValue + React key prop for remount instead of useEffect+setState (ESLint compliance)"
  - "ProductOverrideTable accepts resetKey prop to force DebouncedInput remount after bulk override or clear"
  - "GatewayPlanSelector uses useRef for parent onChange callback to prevent re-render loops (Phase 11 fix pattern)"
  - "handleSaveAndCalculate uses Promise.all for parallel PUT overrides + PUT config, then sequential GET calculate"
  - "Bulk override applies on top of existing overrides (not real prices) unless no override exists yet"
  - "Escenarios link visible to all users (not admin-only) per CONTEXT.md"

patterns_established:
  - "DebouncedInput with key-based remount: uncontrolled input with defaultValue, parent syncs via React key prop change"
  - "Parallel API saves: group independent mutations in Promise.all before dependent calculations"

requirements_completed: [SCEN-01, SCEN-02, SCEN-03, SCEN-04]

metrics:
  duration: 5min
  completed: "2026-03-29T04:46Z"
  tasks: 3
  files: 12
---

# Phase 12 Plan 02: Scenarios Frontend Summary

**Scenarios list page, editor with product override table (debounced inputs), bulk price adjustments, margin comparison with real vs simulated, and sidebar integration using ref-based callback patterns.**

## Performance

- **Duration:** 5 min (code creation by previous executor + verification by continuation)
- **Started:** 2026-03-29T04:12Z
- **Completed:** 2026-03-29T04:46Z
- **Tasks:** 3 (2 code + 1 verification)
- **Files modified:** 12

## Accomplishments

- Scenarios list page at /escenarios with create/delete/toggle-public CRUD, empty state, shared scenarios section
- Scenario editor at /escenarios/[id] with full product override table, debounced inputs, blue highlight for overridden prices
- Bulk override dialog with percentage adjustment by product type, preview count, override-first base pricing
- Margin comparison: real vs simulated with color-coded green/red differences and TrendingUp/TrendingDown icons
- GatewayPlanSelector with ref-based callback pattern (Phase 11 fix) preventing infinite re-render loops
- handleSaveAndCalculate with parallel Promise.all saves then sequential calculate
- Sidebar integration: Escenarios under Herramientas for all users (not admin-only)

## Task Commits

Each task was committed atomically:

1. **Task 1: Types + Leaf Components + Sidebar** - `07e8c27` (feat) - 7 leaf components + sidebar update
2. **Task 2: Pages + Parent Components** - `2057457` (feat) - 4 page/client files with CRUD and editor logic
3. **Task 3: Visual verification** - checkpoint (verification + SUMMARY creation)

## Files Created/Modified

- `nemea-front/src/components/scenarios/types.ts` - Scenario, ScenarioOverride, ScenarioCalcResult, ScenarioCalcResponse, GatewayPlanConfig, CreateScenarioPayload, OverrideItem types
- `nemea-front/src/components/scenarios/ScenarioCard.tsx` - Card with name link, date, override count, gateway, delete button (Trash2), public toggle (Switch), shared badge
- `nemea-front/src/components/scenarios/CreateScenarioDialog.tsx` - Dialog with name input, gateway select, plan select, apiClientFetch POST
- `nemea-front/src/components/scenarios/ProductOverrideTable.tsx` - Editable table with DebouncedInput (300ms, key-based remount), blue highlight for overrides, margin columns with color coding
- `nemea-front/src/components/scenarios/BulkOverrideDialog.tsx` - Dialog with product type select, percentage input, preview count, applies on top of existing overrides
- `nemea-front/src/components/scenarios/GatewayPlanSelector.tsx` - Card with 5 selects (gateway, plan, method, days, installments), ref-based onChange callback, cascading reset on gateway change
- `nemea-front/src/components/scenarios/MarginSummary.tsx` - Card with 3 stats: average margin, products with override, total simulated margin
- `nemea-front/src/app/(app)/escenarios/page.tsx` - RSC page fetching scenarios + config via Promise.all
- `nemea-front/src/app/(app)/escenarios/ScenarioListClient.tsx` - Client component with CRUD, empty state, delete AlertDialog, toggle public, shared scenarios section
- `nemea-front/src/app/(app)/escenarios/[id]/page.tsx` - RSC page fetching scenario + products + config via Promise.all, Next.js 16 async params
- `nemea-front/src/app/(app)/escenarios/[id]/ScenarioEditorClient.tsx` - Editor with override state (Record), gateway config state, useCallback handlers, parallel saves, bulk override, clear overrides
- `nemea-front/src/components/layout/AppSidebar.tsx` - Added LineChart import and Escenarios entry to HERRAMIENTAS_ITEMS

## Decisions Made

- **DebouncedInput pattern**: Used uncontrolled `defaultValue` with React `key` prop for remount (instead of controlled value with useEffect sync) to comply with ESLint exhaustive-deps rule. The resetKey prop forces remount after bulk override or clear operations.
- **Ref-based callback for GatewayPlanSelector**: Follows the Phase 11 fix pattern (11-03) using `useRef(onChange)` with `onChangeRef.current()` calls to prevent the parent callback from being a useEffect dependency and causing infinite re-renders.
- **Parallel saves in handleSaveAndCalculate**: PUT overrides and PUT config run in parallel via Promise.all (2 round-trips instead of 3 sequential), then GET calculate runs sequentially since it depends on saved data.
- **Override-first base pricing**: Bulk adjustment uses `newOverrides[product.id] ?? product.currentPrice` so adjustments stack on existing overrides rather than always resetting to real price.
- **Escenarios visibility**: Available to all users (admin + investor) per CONTEXT.md, not added to ADMIN_ONLY_ROUTES in middleware.ts.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Added resetKey prop to ProductOverrideTable**
- **Found during:** Task 1 (ProductOverrideTable creation)
- **Issue:** DebouncedInput uses uncontrolled defaultValue+key pattern. After bulk override or clear, the parent's overridesState changes but DebouncedInput wouldn't re-render because defaultValue doesn't trigger re-render on existing DOM elements.
- **Fix:** Added `resetKey` prop that increments on bulk override and clear, included in DebouncedInput's key (`${product.id}-${resetKey}`) to force remount.
- **Files modified:** ProductOverrideTable.tsx, ScenarioEditorClient.tsx
- **Commit:** 07e8c27 (Task 1), 2057457 (Task 2)

---

**Total deviations:** 1 auto-fixed (1 missing critical)
**Impact on plan:** Essential for correct DebouncedInput behavior after bulk operations. No scope creep.

## Verification Results

| Check | Result |
|-------|--------|
| TypeScript compilation (tsc --noEmit) | PASS -- 0 errors |
| types.ts Scenario with isPublic | PASS |
| types.ts ScenarioCalcResponse with results | PASS |
| types.ts GatewayPlanConfig with gatewaySlug | PASS |
| ScenarioCard use client + Link + Trash2 + Switch | PASS |
| CreateScenarioDialog placeholder text | PASS |
| ProductOverrideTable DebouncedInput + blue highlight + green/red margins | PASS |
| BulkOverrideDialog title + preview text | PASS |
| GatewayPlanSelector useRef(onChange) + onChangeRef.current | PASS |
| GatewayPlanSelector no onChange in useEffect deps | PASS |
| MarginSummary Card + Margen promedio | PASS |
| AppSidebar LineChart + Escenarios entry | PASS |
| middleware.ts no /escenarios in ADMIN_ONLY_ROUTES | PASS |
| ScenarioListClient empty state + AlertDialog + toggle-public | PASS |
| ScenarioEditorClient useCallback + Promise.all + Math.round | PASS |
| [id]/page.tsx await params + Promise.all | PASS |

## Issues Encountered

None -- plan executed as written with the resetKey enhancement.

## Known Stubs

None -- all components are fully wired to the backend API. No placeholder text or hardcoded empty values.

## User Setup Required

None -- no external service configuration required.

## Next Phase Readiness

- Scenarios frontend complete, ready for Phase 13 (Investor Dashboard)
- Dashboard can import Scenario types and reference /escenarios pages for scenario-aware views
- GatewayPlanSelector pattern (ref-based callback) is reusable for any future cascading select components

## Self-Check: PASSED

All 12 files verified present. Both commit hashes (07e8c27, 2057457) verified in nemea-front git log.

---
*Phase: 12-scenarios*
*Completed: 2026-03-29*
