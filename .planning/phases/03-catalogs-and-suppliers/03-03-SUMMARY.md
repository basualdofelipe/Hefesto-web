---
phase: 03-catalogs-and-suppliers
plan: 03
subsystem: ui
tags: [next.js, shadcn, sidebar, tabs, react-hook-form, zod, tailwind]

# Dependency graph
requires:
  - phase: 03-catalogs-and-suppliers/03-01
    provides: UUID migration for user entity
  - phase: 03-catalogs-and-suppliers/03-02
    provides: Catalog and supplier REST API endpoints
  - phase: 02-auth
    provides: NextAuth session, JWT tokens, useIsAdmin hook
provides:
  - Collapsible sidebar navigation layout (AppSidebar + route groups)
  - Catalog management page with 6 tabs and inline CRUD
  - Supplier list page with search and status toggle
  - Supplier create/edit form pages with zod validation
  - Client-side API wrapper (apiClientFetch) for mutations
affects: [04-supplies, 05-products, 06-costs, 07-expenses]

# Tech tracking
tech-stack:
  added: [shadcn/sidebar, shadcn/table, shadcn/switch, shadcn/badge, shadcn/tooltip, shadcn/dialog, shadcn/sheet, shadcn/skeleton, react-hook-form, zod, @hookform/resolvers]
  patterns: [route-groups, client-api-wrapper, inline-crud, server-component-data-fetch]

key-files:
  created:
    - nemea-front/src/lib/api-client.ts
    - nemea-front/src/components/layout/AppSidebar.tsx
    - nemea-front/src/app/(app)/layout.tsx
    - nemea-front/src/app/(auth)/layout.tsx
    - nemea-front/src/app/(app)/catalogos/page.tsx
    - nemea-front/src/components/catalogs/CatalogTabContent.tsx
    - nemea-front/src/components/catalogs/CatalogItemRow.tsx
    - nemea-front/src/app/(app)/proveedores/page.tsx
    - nemea-front/src/app/(app)/proveedores/nuevo/page.tsx
    - nemea-front/src/app/(app)/proveedores/[id]/editar/page.tsx
    - nemea-front/src/app/(app)/proveedores/[id]/editar/EditSupplierClient.tsx
    - nemea-front/src/components/suppliers/SupplierTable.tsx
    - nemea-front/src/components/suppliers/SupplierForm.tsx
  modified:
    - nemea-front/src/app/layout.tsx
    - nemea-front/src/components/layout/Header.tsx

key-decisions:
  - "Route groups (app)/(auth) separate sidebar layout from auth pages cleanly"
  - "Client-side apiClientFetch wrapper mirrors server-side apiFetch for mutations"
  - "Inline CRUD pattern for catalogs avoids page navigation for simple name edits"
  - "Server-component data fetching with client-component interactivity for catalog tabs"
  - "EditSupplierClient wrapper component bridges server-side fetch and client-side form"

patterns-established:
  - "Route groups: (app) for authenticated pages with sidebar, (auth) for login/denied without sidebar"
  - "Sidebar navigation: AppSidebar with collapsible icon-only mode, SidebarTrigger in Header"
  - "Client mutations: useSession() for token + apiClientFetch for POST/PUT/PATCH/DELETE"
  - "Server data loading: apiFetch in server components, pass initialData to client components"
  - "Inline CRUD: CatalogItemRow display/edit toggle pattern with optimistic local state updates"

requirements-completed: [CATL-01, CATL-02, CATL-03, CATL-04, CATL-05, CATL-06, SUPP-01, SUPP-02]

# Metrics
duration: ~20min
completed: 2026-03-01
---

# Phase 3 Plan 3: Frontend Catalog & Supplier UI Summary

**Shadcn sidebar layout with route groups, 6-tab catalog inline CRUD page, and supplier list/form pages with zod validation**

## Performance

- **Duration:** ~20 min (across multiple sessions with checkpoint)
- **Started:** 2026-03-01
- **Completed:** 2026-03-01
- **Tasks:** 4 (3 auto + 1 checkpoint approved)
- **Files modified:** 29

## Accomplishments

- Collapsible sidebar navigation with Inicio, Catalogos, Proveedores items -- becomes the primary nav pattern for all future phases
- Route groups (app)/(auth) cleanly separate sidebar layout from auth pages
- Catalog management page with 6 tabs (Tipos, Nombres, Terminaciones, Colores, Talles, Tipos de Insumo) and inline CRUD with optimistic updates
- Supplier list with search filtering, active/inactive toggle, and dimmed inactive rows
- Supplier create/edit forms with react-hook-form + zod validation
- Client-side API wrapper (apiClientFetch) for authenticated mutations from client components
- Role-based visibility: admin sees add/edit/delete controls, USER role sees read-only views

## Task Commits

Each task was committed atomically (in nemea-front sub-repo):

1. **Task 1: Install Shadcn + API client + layout restructure** - `3382dfe` (feat)
2. **Task 2: Catalog management page with 6 tabs and inline CRUD** - `ff68d07` (feat)
3. **Task 3: Supplier list, form, create/edit pages** - `d1d9d9e` (feat)
4. **Task 4: Visual verification checkpoint** - approved by user (no commit, checkpoint task)

## Files Created/Modified

### Created
- `nemea-front/src/lib/api-client.ts` - Client-side authenticated fetch wrapper for mutations
- `nemea-front/src/components/layout/AppSidebar.tsx` - Collapsible sidebar with nav items
- `nemea-front/src/app/(app)/layout.tsx` - App layout with SidebarProvider + Header
- `nemea-front/src/app/(auth)/layout.tsx` - Minimal centered layout for login/denied
- `nemea-front/src/app/(auth)/acceso-denegado/page.tsx` - Moved from app/acceso-denegado/
- `nemea-front/src/app/(auth)/login/page.tsx` - Moved from app/login/
- `nemea-front/src/app/(app)/page.tsx` - Moved from app/page.tsx (Inicio)
- `nemea-front/src/app/(app)/catalogos/page.tsx` - Server component with 6-tab catalog page
- `nemea-front/src/components/catalogs/CatalogTabContent.tsx` - Client component for inline CRUD per dimension
- `nemea-front/src/components/catalogs/CatalogItemRow.tsx` - Display/edit row with pencil/trash actions
- `nemea-front/src/app/(app)/proveedores/page.tsx` - Server component for supplier list
- `nemea-front/src/app/(app)/proveedores/nuevo/page.tsx` - Create supplier page
- `nemea-front/src/app/(app)/proveedores/[id]/editar/page.tsx` - Edit supplier server component
- `nemea-front/src/app/(app)/proveedores/[id]/editar/EditSupplierClient.tsx` - Edit form client wrapper
- `nemea-front/src/components/suppliers/SupplierTable.tsx` - Table with search, toggle, actions
- `nemea-front/src/components/suppliers/SupplierForm.tsx` - Form with react-hook-form + zod
- `nemea-front/src/hooks/use-mobile.ts` - Shadcn mobile detection hook
- `nemea-front/src/components/ui/badge.tsx` - Shadcn badge component
- `nemea-front/src/components/ui/dialog.tsx` - Shadcn dialog component
- `nemea-front/src/components/ui/sheet.tsx` - Shadcn sheet component (mobile sidebar)
- `nemea-front/src/components/ui/sidebar.tsx` - Shadcn sidebar component
- `nemea-front/src/components/ui/skeleton.tsx` - Shadcn skeleton component
- `nemea-front/src/components/ui/switch.tsx` - Shadcn switch component
- `nemea-front/src/components/ui/table.tsx` - Shadcn table component
- `nemea-front/src/components/ui/tooltip.tsx` - Shadcn tooltip component

### Modified
- `nemea-front/src/app/layout.tsx` - Removed Header from root layout (moved to (app) layout)
- `nemea-front/src/components/layout/Header.tsx` - Simplified to SidebarTrigger + user menu
- `nemea-front/src/app/(app)/__tests__/page.test.tsx` - Moved with page.tsx

## Decisions Made

- **Route groups over nested layouts:** (app)/(auth) route groups provide clean layout separation without affecting URLs
- **Client API wrapper:** apiClientFetch mirrors server-side apiFetch but uses useSession() token -- avoids duplicating auth logic
- **Inline CRUD for catalogs:** Catalogs are simple name-only entities, so inline editing avoids unnecessary page navigation
- **Server + client split:** Server components fetch initial data, client components handle mutations with optimistic updates
- **EditSupplierClient wrapper:** Bridges server-side data fetching and client-side form interactivity

## Deviations from Plan

None - plan executed as written.

## Known UI Issues

The following issues were identified during visual verification (Task 4 checkpoint) and noted for future resolution. They are NOT blocking and do not affect core functionality:

**1. Supplier email validation may be overly strict**
- The zod schema uses `.email()` which validates format but the form field may behave as required when it should be optional. Needs analysis of the zod schema (`.optional().or(z.literal(''))` pattern), backend DTO, and the user's actual supplier data from Sheets to confirm which fields are truly needed.

**2. Catalog tabs mobile wrapping**
- The 6 catalog tabs (Tipos, Nombres, Terminaciones, Colores, Talles, Tipos de Insumo) wrap awkwardly on mobile screens instead of scrolling horizontally. The TabsList should have `overflow-x-auto` or similar for horizontal scrolling on small screens.

**3. Mobile sidebar backdrop**
- The Sheet overlay for mobile sidebar needs verification that it renders and dismisses correctly. Should be tested more thoroughly on actual mobile viewports.

These items should be captured by the phase verifier for resolution in a polish pass.

## Issues Encountered

None - all 3 implementation tasks completed successfully with passing builds.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Sidebar navigation pattern is established for all future phases -- new pages just need nav items added to AppSidebar
- Client-side API wrapper pattern (apiClientFetch) ready for Phase 4 supply mutations
- Route group structure ready for Phase 4+ pages under (app)/
- All catalog and supplier data is manageable through the UI, fulfilling Phase 3 FK foundation requirements
- Phase 4 (Supplies and Price History) can proceed -- it will add supply pages under (app)/insumos/ and reference catalog dimensions + suppliers

## Self-Check: PASSED

- [x] 03-03-SUMMARY.md exists
- [x] Commit 3382dfe found (Task 1)
- [x] Commit ff68d07 found (Task 2)
- [x] Commit d1d9d9e found (Task 3)

---
*Phase: 03-catalogs-and-suppliers*
*Completed: 2026-03-01*
