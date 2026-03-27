---
phase: 11-calculadora
plan: 02
subsystem: frontend
tags: [calculadora, pricing, tiendanube, forward, inverse, desglose, gateway-selectors]
dependency_graph:
  requires: [CalculadoraModule (11-01), TiendanubeConfigAll types, Product types, apiFetch, apiClientFetch]
  provides: [/calculadora page, CalculadoraClient, DesglosePanel, GatewaySelectors, ModeToggle, ProductSelector]
  affects: [AppSidebar.tsx]
tech_stack:
  added: []
  patterns: [derived-state-cascading-selectors, debounced-api-calls, two-column-responsive-layout]
key_files:
  created:
    - nemea-front/src/components/calculadora/types.ts
    - nemea-front/src/components/calculadora/ModeToggle.tsx
    - nemea-front/src/components/calculadora/ProductSelector.tsx
    - nemea-front/src/components/calculadora/GatewaySelectors.tsx
    - nemea-front/src/components/calculadora/DesglosePanel.tsx
    - nemea-front/src/app/(app)/calculadora/CalculadoraClient.tsx
    - nemea-front/src/app/(app)/calculadora/page.tsx
  modified:
    - nemea-front/src/components/layout/AppSidebar.tsx
decisions:
  - Used derived state (useMemo) instead of setState-in-useEffect for cascading gateway selectors (React lint compliance)
  - Products grouped by type.name in Select with SelectGroup for readability
  - Debounce 300ms on calculation trigger to avoid excessive API calls
  - Plan override toggle hidden by default, inherits esencial plan from config
metrics:
  duration: 5min
  completed: 2026-03-27
  tasks: 1/1 (auto) + 1 checkpoint
  files_created: 7
  files_modified: 1
---

# Phase 11 Plan 02: Calculadora Frontend Summary

Full /calculadora page with forward/inverse pricing simulation, cascading gateway selectors, product cost auto-population from DB, and 16-value desglose panel with ARS formatting.

## What Was Built

### Task 1: Types + Components + CalculadoraClient + page.tsx + sidebar link (f6a9345)

**types.ts** -- Frontend interfaces matching backend DTOs:
- `CalcResult` with all 16 intermediate calculation fields
- `CalcInverseResult` extending CalcResult with `precioVenta`
- `CalcBatchItem` for future batch/dashboard use

**ModeToggle.tsx** -- Forward/inverse toggle using Shadcn Tabs:
- "Precio -> Ganancia" (forward) and "Ganancia -> Precio" (inverse)
- Controlled component with mode/onModeChange props

**ProductSelector.tsx** -- Product selection with grouped display:
- Groups active products by `type.name` using Shadcn Select with SelectGroup
- Shows product cost inline after selection (formatted with `formatCost`)
- Warns "Sin costo definido" with amber AlertTriangle when cost is null

**GatewaySelectors.tsx** -- Cascading selectors (5 fields):
1. Pasarela (gateway) -- filters active gateways from config
2. Medio de pago -- derived from rates matching selected gateway
3. Tiempo de retiro -- derived from rates matching gateway + method
4. Cuotas -- from active installments in config
5. Plan TN -- hidden behind "Simular otro plan" switch, defaults to 'esencial'

Uses derived state pattern (`useMemo` for `effectiveMethod` and `effectiveDays`) instead of setState-in-useEffect to comply with React Compiler's `react-hooks/set-state-in-effect` rule. When gateway changes, downstream selections reset and derivation picks first available option.

**DesglosePanel.tsx** -- Right column with all 16 intermediate values:
- Inverse mode: "Precio de venta necesario" highlighted in blue at top
- Deductions in red: comision pasarela (with tasa breakdown), financiacion cuotas (conditional), CPT Tiendanube, retenciones IIBB
- IVA breakdown: debito fiscal (negative), credito producto (positive, indented), credito comision (positive, indented), IVA neto
- Costo producto + IVA
- GANANCIA REAL with margen % highlighted in green with large font
- ARS formatting via `Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' })`
- Disclaimer text at bottom

**CalculadoraClient.tsx** -- Main client component (323 lines):
- Two-column responsive layout: `grid grid-cols-1 lg:grid-cols-2 gap-6`
- State management: mode, selectedProduct, precioVenta, costoEnvio, gananciaDeseada, result, loading, gatewayConfig
- Debounced calculation (300ms) via useRef + useCallback with cleanup on unmount
- Forward mode: POST /api/calculadora/forward with productId + precioVenta + gateway config
- Inverse mode: POST /api/calculadora/inverse with productId + gananciaDeseada + gateway config
- Backend ResponseInterceptor wraps in `{ data: T }` -- frontend unwraps `.data`
- Error handling: toast.error() from sonner for 400 responses (inverse edge cases)
- Loading state with Loader2 spinner
- Empty state: "Selecciona un producto y completa los datos"
- Input fields with $ prefix using Shadcn Input wrapped in Cards

**page.tsx** -- Server component (25 lines):
- Parallel fetch: products (GET /api/products) + TN config (GET /api/tiendanube-config/all)
- Both ADMIN and USER can access (no role redirect)
- Passes `productsRes.data` and `configRes.data` to CalculadoraClient

**AppSidebar.tsx** -- Added "Herramientas" section:
- New `HERRAMIENTAS_ITEMS` array with Calculator icon from lucide-react
- Rendered as SidebarGroup with label "Herramientas" between Finanzas and Productos
- Outside `isAdmin` conditional -- visible to ALL users

### Task 2: Checkpoint (human-verify)

**What to verify:**
1. Start dev servers (backend + frontend)
2. Login as admin, navigate to /calculadora (sidebar "Herramientas" section)
3. Forward mode: select product with cost, enter precio de venta 87000, envio 7315, select Pago Nube > Tarjeta > 14 dias > 1 cuota -- verify full desglose with green ganancia real
4. Inverse mode: toggle to "Ganancia -> Precio", enter ganancia deseada 50000 -- verify blue "Precio de venta necesario" and desglose
5. Edge case: enter extremely high ganancia (99999999) in inverse -- verify toast error (not broken UI)
6. Gateway cascade: switch to Mercado Pago -- verify method and withdrawal options change
7. Plan override: toggle "Simular otro plan", change plan -- verify recalculation
8. Mobile: resize to mobile width -- verify single-column stacked layout

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed setState-in-useEffect lint errors in GatewaySelectors**
- **Found during:** Task 1 verification (npm run lint)
- **Issue:** Initial implementation used useEffect to reset paymentMethod and withdrawalDays when upstream selectors changed, triggering `react-hooks/set-state-in-effect` lint errors
- **Fix:** Refactored to derived state pattern using `useMemo` for `effectiveMethod` and `effectiveDays` -- values are computed from available options rather than being set via setState in effects
- **Files modified:** nemea-front/src/components/calculadora/GatewaySelectors.tsx
- **Commit:** f6a9345 (included in same commit)

## Verification Results

1. `npx tsc --noEmit` -- PASSED (zero type errors)
2. `npm run lint` -- PASSED (zero errors, 2 pre-existing warnings in unrelated files)
3. `npx prettier --check` -- PASSED (all files formatted)
4. File counts: page.tsx 25 lines (min 15), CalculadoraClient.tsx 323 lines (min 100), DesglosePanel.tsx 197 lines (min 50), GatewaySelectors.tsx 258 lines (min 40)

## Known Stubs

None -- all components fully implemented with real data sources wired.

## Self-Check: PASSED

All 7 created files and 1 modified file verified on disk. Commit f6a9345 verified in git log.
