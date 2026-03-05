---
phase: 04-supplies-and-price-history
verified: 2026-03-05T23:10:00Z
status: passed
score: 7/7 must-haves verified
re_verification:
  previous_status: gaps_found
  previous_score: 4/7
  gaps_closed:
    - "Admin can toggle supply active/inactive status (with supplier validation)"
    - "Next.js dev server runs without compilation/runtime errors"
    - "Admin can edit a supply via modal and see updated data when reopening"
  gaps_remaining: []
  regressions: []
human_verification:
  - test: "Full CRUD workflow on /insumos page"
    expected: "Create, expand, edit, add price, view history, toggle status all work end-to-end"
    why_human: "Requires running dev servers and visual interaction"
  - test: "Supplier cascade deactivation reflects in supply list"
    expected: "Deactivating a supplier in /proveedores makes its supplies inactive in /insumos"
    why_human: "Cross-page workflow requiring navigation and state verification"
---

# Phase 4: Supplies and Price History Verification Report

**Phase Goal:** Supplies with their types and suppliers exist, and every price change is preserved as an immutable historical record. CRUD insumos con tipo, proveedor, unidad, notas. Historial de precios append-only. Precio actual = ultimo registro. Soft delete con cascade desde proveedor. Frontend: tabla agrupada por tipo, expandible, modals, precio inline.
**Verified:** 2026-03-05T23:10:00Z
**Status:** passed
**Re-verification:** Yes -- after gap closure (plan 04-03)

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Admin can create a supply by selecting its type and supplier | VERIFIED | POST /supplies endpoint with CreateSupplyDto (typeId, supplierId, unitType, notes, initialPrice). Frontend SupplyFormDialog with create mode. Transaction for atomic supply+price creation. |
| 2 | Admin can add a new price to a supply; previous price preserved in history | VERIFIED | POST /supplies/:id/prices endpoint. AddPriceInline component calls API. SupplyPriceHistory is append-only (no update/delete endpoints). |
| 3 | User can view all historical prices for any supply sorted newest to oldest | VERIFIED | GET /supplies/:id/prices returns `order: { createdAt: 'DESC' }`. PriceHistoryDialog fetches and renders date+amount list. |
| 4 | Current price is always the most recently added record | VERIFIED | DISTINCT ON (supply_id) ORDER BY created_at DESC query in findAll(). findOne() uses findOne with order createdAt DESC. Both return currentPrice mapped onto supply. |
| 5 | Admin can toggle supply active/inactive status (with supplier validation) | VERIFIED | toggleStatus() at line 204 loads supply with supplier relation. Lines 214-219 check `!supply.isActive && !supply.supplier.isActive` and throw ConflictException before allowing reactivation. Fixed in commit 3117d49. |
| 6 | Next.js dev server runs without errors | VERIFIED | Root cause was missing Fragment key in SupplyTypeGroup.tsx map rendering. Fixed by importing Fragment and using `<Fragment key={supply.id}>` at line 81. Fixed in commit fc23d20. |
| 7 | Edit modal shows fresh data when reopened after editing | VERIFIED | SupplyFormDialog.tsx lines 89-109: useEffect watching [supply, open, reset] calls reset() with current supply values on every dialog open. Handles both prop changes and cancel-then-reopen. Fixed in commit fc23d20. |

**Score:** 7/7 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/supplies/entities/supply.entity.ts` | Supply entity with ManyToOne to SupplyType/Supplier, UnitType enum, is_active | VERIFIED | Extends BaseEntity, ManyToOne eager to both, UnitType enum exported |
| `nemea-back/src/supplies/entities/supply-price-history.entity.ts` | Append-only price history with ManyToOne to Supply | VERIFIED | decimal(12,2), onDelete CASCADE |
| `nemea-back/src/supplies/supplies.service.ts` | CRUD + toggle + price history + DISTINCT ON + supplier validation | VERIFIED | 268 lines, all methods present, transaction for create, DISTINCT ON query, deactivateBySupplier, ConflictException guard on reactivation |
| `nemea-back/src/supplies/supplies.controller.ts` | 7 REST endpoints with Swagger and roles | VERIFIED | All 7 endpoints, @Roles(Role.ADMIN) on mutations, ParseUUIDPipe |
| `nemea-back/src/supplies/supplies.module.ts` | Module with repos and exports | VERIFIED | Registered in app.module.ts |
| `nemea-back/src/supplies/dto/create-supply.dto.ts` | CreateSupplyDto with validators | VERIFIED | Exists with validators |
| `nemea-back/src/supplies/dto/update-supply.dto.ts` | UpdateSupplyDto (no supplierId) | VERIFIED | Uses OmitType to exclude supplierId/initialPrice |
| `nemea-back/src/supplies/dto/create-supply-price.dto.ts` | Price DTO | VERIFIED | Exists |
| `nemea-back/src/database/migrations/1772400000000-CreateSuppliesAndPriceHistory.ts` | Raw SQL migration | VERIFIED | File exists |
| `nemea-front/src/app/(app)/insumos/page.tsx` | Server component fetching supplies, types, suppliers | VERIFIED | Promise.all 3 fetches, passes to SupplyTable |
| `nemea-front/src/components/supplies/SupplyTable.tsx` | Client component with search, filters, grouped rendering | VERIFIED | Search, supplier filter, showInactive toggle, grouped by type |
| `nemea-front/src/components/supplies/SupplyTypeGroup.tsx` | Collapsible section per type with Fragment key | VERIFIED | 127 lines, Collapsible, count badge, expandable rows, Fragment key on line 81 |
| `nemea-front/src/components/supplies/SupplyExpandedRow.tsx` | Inline expanded with details and actions | VERIFIED | Notas, supplier link, price, toggle, edit/price/history buttons |
| `nemea-front/src/components/supplies/SupplyFormDialog.tsx` | Create/edit modal with zod and form reset | VERIFIED | 340 lines, form with useEffect reset on [supply, open, reset] deps |
| `nemea-front/src/components/supplies/PriceHistoryDialog.tsx` | Price history modal | VERIFIED | Fetches on open, date+amount list, loading skeleton |
| `nemea-front/src/components/supplies/AddPriceInline.tsx` | Inline price input | VERIFIED | Number input, confirm/cancel, POST to API |
| `nemea-front/src/components/supplies/types.ts` | Shared types and helpers | VERIFIED | Supply, PriceRecord, UNIT_LABELS, formatPrice |
| `nemea-front/src/components/layout/AppSidebar.tsx` | Sidebar with Datos base group | VERIFIED | Has "Datos base" SidebarGroupLabel with Insumos nav item |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| supplies.service.ts | Supply + SupplyPriceHistory | @InjectRepository | WIRED | Both repos injected, used in all CRUD methods |
| supplies.service.ts | Transaction for create+price | entityManager.transaction | WIRED | Lines 116-139, atomic supply+price creation |
| suppliers.service.ts | Supply repo for cascade | @InjectRepository(Supply) | WIRED | update isActive=false on supplier deactivation |
| supplies.service.ts | DISTINCT ON latest price | raw SQL query | WIRED | Lines 52-56, batch price fetch |
| supplies.service.ts | ConflictException on reactivation | supply.supplier.isActive check | WIRED | Lines 214-219, throws if supplier inactive |
| insumos/page.tsx | /api/supplies | apiFetch server-side | WIRED | Fetches with includeInactive=true |
| SupplyTable.tsx | SupplyTypeGroup | Component rendering | WIRED | Maps grouped supplies |
| SupplyFormDialog.tsx | /api/supplies | apiClientFetch POST/PUT | WIRED | Create and edit mutations |
| SupplyFormDialog.tsx | useEffect reset | [supply, open, reset] deps | WIRED | Lines 89-109, resets form on dialog open |
| AddPriceInline.tsx | /api/supplies/:id/prices | apiClientFetch POST | WIRED | Posts price |
| PriceHistoryDialog.tsx | /api/supplies/:id/prices | apiClientFetch GET | WIRED | Fetches history on open |
| SuppliesModule | app.module.ts | Module import | WIRED | Registered in app.module.ts |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| SPPL-01 | 04-01, 04-02 | Admin can CRUD supplies with name, type, and supplier | SATISFIED | Full CRUD endpoints + frontend create/edit/toggle |
| SPPL-02 | 04-01, 04-02 | Admin can add a new price (creates historical record, previous preserved) | SATISFIED | POST /supplies/:id/prices, append-only entity, AddPriceInline component |
| SPPL-03 | 04-01, 04-02 | User can view price history for any supply | SATISFIED | GET /supplies/:id/prices, PriceHistoryDialog |
| SPPL-04 | 04-01 | Current price is always the most recent record | SATISFIED | DISTINCT ON query, findOne with order DESC |
| SPPL-05 | 04-01, 04-02, 04-03 | Admin can deactivate/reactivate supplies (soft delete) | SATISFIED | Toggle endpoint with ConflictException guard for supplier validation |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none) | - | - | - | All previously flagged anti-patterns resolved in plan 04-03 |

### Human Verification Required

### 1. Full CRUD Workflow

**Test:** Navigate to /insumos, create a supply, expand it, edit it, add a price, view history, toggle status.
**Expected:** All operations complete successfully with appropriate toast messages and UI updates.
**Why human:** Requires running dev servers and visual interaction with the app.

### 2. Supplier Cascade Verification

**Test:** Deactivate a supplier in /proveedores that has active supplies. Navigate to /insumos with "Mostrar inactivos" on.
**Expected:** The supplier's supplies should appear as inactive. Attempting to reactivate one should show conflict error.
**Why human:** Cross-page workflow requiring navigation and state verification.

### Gaps Summary

All 3 gaps from the initial verification have been closed:

1. **Supplier validation on reactivation (was BLOCKER):** ConflictException guard added in `toggleStatus()` at lines 214-219 of `supplies.service.ts`. Checks `!supply.isActive && !supply.supplier.isActive` before allowing toggle. Commit `3117d49`.

2. **Dev server console error (was FAILED):** Root cause identified as missing Fragment key in `SupplyTypeGroup.tsx` `.map()` call. Fixed by importing `Fragment` and using `<Fragment key={supply.id}>`. Commit `fc23d20`.

3. **Stale edit modal data (was FAILED):** `useEffect` added in `SupplyFormDialog.tsx` watching `[supply, open, reset]` that calls `reset()` with current values on every dialog open. Handles prop changes and cancel-then-reopen scenarios. Commit `fc23d20`.

Phase 04 goal fully achieved. All 5 requirements (SPPL-01 through SPPL-05) satisfied.

---

_Verified: 2026-03-05T23:10:00Z_
_Verifier: Claude (gsd-verifier)_
