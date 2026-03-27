---
phase: 10-tiendanube-config
plan: 02
subsystem: frontend
tags: [tiendanube, config, admin-page, rates, accordion, plan-selector]
dependency_graph:
  requires: [TiendanubeConfigModule (10-01)]
  provides: [/configuracion/tiendanube page, plan selector, gateway rate editing, CPT editing, installment editing, tax editing, sidebar link]
  affects: [AppSidebar.tsx, middleware.ts]
tech_stack:
  added: [shadcn-accordion]
  patterns: [RSC-admin-page, apiClientFetch-PUT-per-row, plan-selector-state-lift, compound-component-header]
key_files:
  created:
    - nemea-front/src/components/tiendanube-config/types.ts
    - nemea-front/src/app/(app)/configuracion/tiendanube/page.tsx
    - nemea-front/src/app/(app)/configuracion/tiendanube/TiendanubeConfigClient.tsx
    - nemea-front/src/components/tiendanube-config/GatewaySection.tsx
    - nemea-front/src/components/tiendanube-config/PlansSection.tsx
    - nemea-front/src/components/tiendanube-config/InstallmentsSection.tsx
    - nemea-front/src/components/tiendanube-config/TaxConfigSection.tsx
    - nemea-front/src/components/ui/accordion.tsx
  modified:
    - nemea-front/src/components/layout/AppSidebar.tsx
    - nemea-front/src/middleware.ts
decisions:
  - Compound component pattern for GatewaySection (Header as static property) to separate accordion trigger content from body
  - Plan selector state lifted to TiendanubeConfigClient, passed down to GatewaySection headers and PlansSection
  - Local state updates after PUT calls (onPlanUpdated callback) instead of router.refresh for immediate feedback
  - All accordion sections expanded by default via defaultValue array
metrics:
  duration: 7min
  completed: 2026-03-27
  tasks: 2/2
  files_created: 8
  files_modified: 2
---

# Phase 10 Plan 02: Tiendanube Config Frontend Summary

Admin config page at /configuracion/tiendanube with plan selector dropdown, collapsible gateway/plans/installments/taxes sections, per-row save buttons calling backend PUT endpoints, and sidebar navigation link.

## What Was Built

### Task 1: Config page with all rate editing components (e9709fc)

Created the full frontend for Tiendanube configuration:

1. **types.ts** - All TypeScript interfaces matching backend contract. All rate fields typed as `number` (not `string`). Includes `PAYMENT_METHOD_LABELS` map for human-readable display.

2. **page.tsx (RSC)** - Server component following `/usuarios/page.tsx` pattern: `auth()` check with admin role redirect, `apiFetch` to `/api/tiendanube-config/all`, passes data to client component.

3. **TiendanubeConfigClient.tsx** - Main client component with:
   - Plan selector dropdown (Shadcn Select) defaulting to "Esencial"
   - Rates grouped by gateway via `useMemo` Map
   - Shadcn Accordion (`type="multiple"`, all sections expanded by default)
   - 3 gateway sections + plans + installments + taxes
   - "Verificar tasas en Pago Nube" external link at bottom

4. **GatewaySection.tsx** - Per-gateway rate table with editable Input fields, per-row Guardar button calling PUT `/api/tiendanube-config/gateway-rates/:gatewayId`. Shows CPT info from selected plan in header. Inactive gateway badge "Pendiente".

5. **PlansSection.tsx** - CPT editing per Tiendanube plan. Editable cptPagoNube and cptOtherGateways fields. "CPT Otros" disabled for Inicial plan (onlyPagoNube=true). Calls PUT `/api/tiendanube-config/plans/:id`. Updates parent state via `onPlanUpdated` callback.

6. **InstallmentsSection.tsx** - Installment rate editing with per-row save calling PUT `/api/tiendanube-config/installment-rates/:installments`.

7. **TaxConfigSection.tsx** - IVA and IIBB/SIRTAC rate editing with single Guardar button calling PUT `/api/tiendanube-config/taxes`. Includes note about consulting accountant.

8. **accordion.tsx** - Shadcn Accordion component installed and formatted for project conventions.

Also added `/configuracion` to `ADMIN_ONLY_ROUTES` in middleware.ts for server-side route protection (pre-mortem catch: not in original plan but required for security consistency).

### Task 2: Sidebar link + full verification (a2f9d75)

Added "Config Tiendanube" with Settings icon to `ADMIN_ITEMS` in AppSidebar. Visible only to admin users (inherits existing `{isAdmin && ...}` conditional). Verified full TypeScript compilation and lint pass.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Security] Added /configuracion to ADMIN_ONLY_ROUTES**
- **Found during:** Task 1
- **Issue:** The plan did not mention adding /configuracion to middleware.ts ADMIN_ONLY_ROUTES, but without it non-admin users could bypass the RSC redirect by navigating directly
- **Fix:** Added '/configuracion' to ADMIN_ONLY_ROUTES array
- **Files modified:** nemea-front/src/middleware.ts
- **Commit:** e9709fc

**2. [Rule 3 - Blocking] Fixed Shadcn Accordion double quotes**
- **Found during:** Task 1
- **Issue:** Shadcn generated accordion.tsx with double quotes, failing project lint (single quotes required)
- **Fix:** Ran prettier --write on the file
- **Files modified:** nemea-front/src/components/ui/accordion.tsx
- **Commit:** e9709fc

## Verification Results

1. `npx tsc --noEmit` - PASSED (zero errors)
2. `npm run lint` - PASSED (zero errors, 2 pre-existing warnings in unrelated files)
3. Route exists: `nemea-front/src/app/(app)/configuracion/tiendanube/page.tsx` - CONFIRMED
4. Sidebar has "Config Tiendanube" in ADMIN_ITEMS - CONFIRMED (1 occurrence)
5. Page has admin-only redirect (session role check) - CONFIRMED
6. Plan selector renders all 4 plans and defaults to Esencial - CONFIRMED (5 references to selectedPlan in client)
7. PlansSection.tsx exists with editable CPT fields and Guardar buttons - CONFIRMED
8. PUT /api/tiendanube-config/plans/:id called on CPT save - CONFIRMED
9. GatewaySection shows CPT info from selected plan in header - CONFIRMED
10. "Verificar tasas" link points to official Pago Nube URL with target="_blank" - CONFIRMED
11. All save buttons call correct PUT endpoints via apiClientFetch - CONFIRMED
12. Types file exports all interfaces with `number` for rate fields - CONFIRMED (8 number-typed rate fields)

## Known Stubs

None - all components are fully wired to backend API endpoints from Plan 10-01. Data flows from RSC fetch through client components. Save operations call real PUT endpoints.

## Self-Check: PASSED

All 8 created files verified on disk. All 2 modified files verified. Both commit hashes (e9709fc, a2f9d75) found in git log. SUMMARY.md exists at expected path.
