---
phase: 07-expenses-and-config
verified: 2026-03-11T00:00:00Z
status: human_needed
score: 19/19 must-haves verified (automated)
human_verification:
  - test: "End-to-end expense CRUD flow"
    expected: "Admin creates, edits, and deletes an expense successfully. USER role sees list but no create/edit/delete buttons."
    why_human: "Role-based button visibility and real-time state updates require browser interaction to confirm."
  - test: "Month grouping and subtotals accuracy"
    expected: "Each collapsible month header shows correct count and sum of expenses in that month."
    why_human: "Arithmetic correctness of subtotals and month label formatting (Spanish locale) cannot be confirmed without rendering."
  - test: "Combined category + date range filter"
    expected: "Selecting a category and changing date range re-fetches and shows only matching expenses; summary bar updates."
    why_human: "Dynamic re-fetch behavior and summary bar reactivity require browser interaction."
  - test: "FK constraint block on category delete"
    expected: "Attempting to delete an expense category that has linked expenses returns an error (HTTP 409 or 500 with FK message), not a silent success."
    why_human: "Error handling path depends on DB-level FK rejection and frontend error toast, not verifiable statically."
  - test: "Expense categories tab on /catalogos"
    expected: "'Categorias de Gasto' tab appears on /catalogos and allows inline create/edit/delete via CatalogTabContent."
    why_human: "Tab rendering and interaction via existing generic component need visual confirmation."
  - test: "Sidebar order"
    expected: "Sidebar shows: Inicio > Finanzas (Gastos) > Productos > Datos base (Catalogos, Proveedores, Insumos)"
    why_human: "Visual layout and order of sidebar groups requires browser rendering to confirm."
---

# Phase 7: Expenses and Config — Verification Report

**Phase Goal:** Expense tracking with categories, CRUD, filtering by month/category, and role-based access. Closes all v1 backend + frontend features.
**Verified:** 2026-03-11
**Status:** human_needed
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths — Plan 07-01 (Backend)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | POST /api/expenses creates an expense with amount, concept, date, and category FK | VERIFIED | `ExpensesController.create` calls `ExpensesService.create` which validates category, creates entity, returns saved expense |
| 2 | GET /api/expenses returns expenses ordered by date DESC, filterable by categoryId/dateFrom/dateTo | VERIFIED | `ExpensesService.findAll` uses QueryBuilder with `orderBy date DESC`, optional `andWhere` clauses for all three filters |
| 3 | PUT /api/expenses/:id updates an expense | VERIFIED | `ExpensesController.update` with `@Roles(Role.ADMIN)`, calls `ExpensesService.update` with partial field merging |
| 4 | DELETE /api/expenses/:id hard-deletes an expense | VERIFIED | `ExpensesService.remove` calls `repo.remove(expense)` — no soft-delete flag present |
| 5 | GET /api/catalogs/expense-categories returns all expense categories | VERIFIED | `'expense-categories'` added to `VALID_DIMENSIONS` and `dimensionMap` in `CatalogsService`, wired to `expenseCategoryRepo` |
| 6 | POST /api/catalogs/expense-categories creates a new category (ADMIN only) | VERIFIED | `CatalogsController` DIMENSION_ENUM includes `'expense-categories'`; existing generic CRUD enforces ADMIN role for write operations |
| 7 | DELETE /api/catalogs/expense-categories/:id is blocked if category has expenses (FK 23503) | VERIFIED | FK constraint `FK_expenses_category` defined in migration without ON DELETE CASCADE — DB will reject with FK violation |
| 8 | 6 seed categories exist after migration: Materia prima, Packaging, Envio, Herramientas, Servicios, Otros | VERIFIED | Migration `1772600000000` seeds all 6 names in single INSERT statement |

### Observable Truths — Plan 07-02 (Frontend)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Admin can create an expense via modal with amount, concept, date (defaults today), and category dropdown | VERIFIED | `ExpenseFormDialog` — zod schema, react-hook-form, default date `new Date().toISOString().split('T')[0]`, POST via `apiClientFetch` |
| 2 | Admin can edit an expense via same modal prefilled with existing values | VERIFIED | `useEffect` in `ExpenseFormDialog` resets form with `expense.amount`, `expense.concept`, `expense.date`, `expense.category.id` |
| 3 | Admin can delete an expense with confirmation dialog (hard delete) | VERIFIED | `DeleteExpenseDialog` uses AlertDialog, DELETE request via `apiClientFetch` to `/api/expenses/:id` |
| 4 | Expense list shows entries grouped by month with collapsible sections | VERIFIED | `ExpenseTable` calls `groupByMonth`, renders `ExpenseMonthGroup` per entry; `Collapsible` with `defaultOpen` used |
| 5 | Each month header shows month name + expense count + subtotal | VERIFIED | `ExpenseMonthGroup` header: `formatMonthLabel`, `expenses.length`, sum via `reduce(parseFloat(amount))` + `formatAmount` |
| 6 | Summary bar shows grand total + count for filtered period, updates dynamically | VERIFIED | `useMemo` in `ExpenseTable` computes `totalAmount` and `totalCount` from current `expenses` state |
| 7 | Category dropdown filter and date range filter work together | VERIFIED | `handleCategoryChange`, `handleDateFromChange`, `handleDateToChange` all call `fetchExpenses` with current state of all three params |
| 8 | Default view shows current month | VERIFIED | `getCurrentMonthRange()` called in both `page.tsx` (server fetch) and `ExpenseTable` state initialization |
| 9 | Each expense row shows Fecha, Concepto, Categoria (colored badge), Monto | VERIFIED | `ExpenseMonthGroup` table: `formatDateDisplay(date)`, `concept`, badge with `getCategoryColor`, `formatAmount(amount)` |
| 10 | Expense categories tab on /catalogos page works with inline CRUD | VERIFIED | `{ key: 'expense-categories', label: 'Categorias de Gasto' }` added to `DIMENSIONS` array in `catalogos/page.tsx` |
| 11 | Sidebar shows Finanzas group with Gastos item in correct order | VERIFIED | `FINANZAS_ITEMS = [{ label: 'Gastos', href: '/finanzas/gastos', icon: Receipt }]` in `AppSidebar.tsx`; rendered as `SidebarGroup` with label "Finanzas" |
| 12 | USER role can view expenses but cannot create/edit/delete | VERIFIED | `isAdmin` prop passed to `ExpenseTable` and `ExpenseMonthGroup`; `+ Nuevo gasto` button and row action buttons conditionally rendered with `{isAdmin && ...}` |

**Score:** 19/19 truths verified (automated)

---

## Required Artifacts

| Artifact | Provides | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/expenses/expenses.module.ts` | ExpensesModule registered in AppModule | VERIFIED | Imports `[Expense, ExpenseCategory]`, registered in `app.module.ts` line 36 |
| `nemea-back/src/expenses/expenses.controller.ts` | CRUD endpoints for expenses | VERIFIED | Full CRUD: GET, POST, PUT, DELETE with proper decorators and role guards |
| `nemea-back/src/expenses/expenses.service.ts` | Expense business logic with QueryBuilder filtering | VERIFIED | All 5 methods implemented and non-trivial: findAll (QB), findOne, create, update, remove |
| `nemea-back/src/expenses/entities/expense.entity.ts` | Expense entity with amount, concept, date, category FK | VERIFIED | decimal(12,2) amount, varchar(500) concept, DATE date, ManyToOne to ExpenseCategory |
| `nemea-back/src/catalogs/entities/expense-category.entity.ts` | ExpenseCategory entity for catalog dimension | VERIFIED | varchar(100) name with `unique: true`, extends BaseEntity |
| `nemea-back/src/database/migrations/1772600000000-CreateExpenseCategoriesAndExpenses.ts` | Migration creating both tables + seed data + date index | VERIFIED | Creates `expense_categories`, seeds 6 rows, creates `expenses` with FK, adds `IDX_expenses_date` |
| `nemea-front/src/app/(app)/finanzas/gastos/page.tsx` | Expenses page with server-side data fetching | VERIFIED | Server component, fetches categories + current-month expenses in parallel, passes to ExpenseTable |
| `nemea-front/src/components/expenses/ExpenseTable.tsx` | Client component: filters + month groups + summary bar | VERIFIED | 233 lines, substantive implementation with state, useMemo, fetchExpenses, all dialogs wired |
| `nemea-front/src/components/expenses/ExpenseMonthGroup.tsx` | Collapsible month group with rows | VERIFIED | Collapsible with table, subtotal, row-level edit/delete buttons |
| `nemea-front/src/components/expenses/ExpenseFormDialog.tsx` | Create/edit expense modal | VERIFIED | react-hook-form + zod, useEffect reset, POST/PUT via apiClientFetch |
| `nemea-front/src/components/expenses/DeleteExpenseDialog.tsx` | Confirmation dialog for hard delete | VERIFIED | AlertDialog, DELETE request, loading state |
| `nemea-front/src/components/expenses/types.ts` | Expense and ExpenseCategory interfaces, CATEGORY_COLORS, helpers | VERIFIED | All helpers present: getCategoryColor, formatAmount, formatMonthLabel, formatDateDisplay, groupByMonth, getCurrentMonthRange |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `expenses.service.ts` | `expense.entity.ts` | `@InjectRepository(Expense)` | VERIFIED | `@InjectRepository(Expense) private readonly expenseRepo` line 13-14 |
| `catalogs.service.ts` | `expense-category.entity.ts` | dimensionMap entry `'expense-categories'` | VERIFIED | `'expense-categories': this.expenseCategoryRepo` in VALID_DIMENSIONS and dimensionMap |
| `app.module.ts` | `expenses.module.ts` | Module imports | VERIFIED | `import { ExpensesModule }` line 9, `ExpensesModule` in imports array line 36 |
| `ExpenseTable.tsx` | `/api/expenses` | `apiClientFetch` for mutations, URL construction for GET | VERIFIED | `fetchExpenses` uses `apiClientFetch`, form dialog uses `apiClientFetch('/api/expenses', ...)` |
| `catalogos/page.tsx` | `/api/catalogs/expense-categories` | DIMENSIONS array entry | VERIFIED | `{ key: 'expense-categories', label: 'Categorias de Gasto' }` triggers `apiFetch` in map |
| `AppSidebar.tsx` | `/finanzas/gastos` | FINANZAS_ITEMS nav array | VERIFIED | `href: '/finanzas/gastos'` in FINANZAS_ITEMS, rendered via `renderNavItems` |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| EXPN-01 | 07-01, 07-02 | Admin can register an expense with amount, concept, date, and category | SATISFIED | Backend: `POST /api/expenses` with `CreateExpenseDto`. Frontend: `ExpenseFormDialog` with all required fields |
| EXPN-02 | 07-01, 07-02 | Admin can view list of expenses filtered by category or date | SATISFIED | Backend: `GET /api/expenses` with `QueryExpensesDto` (categoryId, dateFrom, dateTo). Frontend: filter controls in `ExpenseTable` that re-fetch on change |
| EXPN-03 | 07-01, 07-02 | Categories include: materia prima, packaging, envío, herramientas, servicios, otros | SATISFIED | Migration seeds all 6 categories. `CATEGORY_COLORS` in `types.ts` maps all 6 names with distinct colors |

No orphaned requirements — all three EXPN requirements are claimed by both plans and have implementation evidence.

---

## Anti-Patterns Found

None. No TODO/FIXME/XXX/HACK markers, no placeholder returns, no empty handlers. All `placeholder` occurrences are standard HTML attribute values on input elements.

---

## Human Verification Required

### 1. End-to-End Expense CRUD Flow

**Test:** Log in as ADMIN. Navigate to /finanzas/gastos. Click "+ Nuevo gasto". Fill in amount (e.g. 1500), concept ("Cuero vaca"), today's date, category "Materia prima". Save. Verify it appears in the list under the current month. Click edit (pencil icon), change amount to 2000, save. Verify update reflected. Click delete (trash icon), confirm. Verify row removed.
**Expected:** All three operations succeed with toast notifications. List updates after each without page reload.
**Why human:** Mutation success, toast behavior, and optimistic state update cannot be confirmed statically.

### 2. USER Role View-Only Access

**Test:** Log in as a USER (non-admin) role. Navigate to /finanzas/gastos.
**Expected:** Expense list renders but no "+ Nuevo gasto" button is visible and no edit/delete icons appear on rows.
**Why human:** Role-based conditional rendering depends on session data at runtime.

### 3. Month Grouping and Subtotals

**Test:** Create 3 expenses — 2 in the current month (e.g. $1000 + $500) and 1 in a previous month ($300). Set date range to include both months.
**Expected:** Two collapsible groups appear. Current month shows "2 gastos" and "$1.500". Previous month shows "1 gasto" and "$300".
**Why human:** `es-AR` locale formatting and `formatMonthLabel` (Spanish month name, capitalized) need visual confirmation.

### 4. Combined Filters

**Test:** With several expenses across categories, select "Packaging" from the category dropdown. Then change the date range.
**Expected:** Each filter change re-fetches from backend. Summary bar total and count update to reflect only the filtered results.
**Why human:** Dynamic re-fetch and summary bar reactivity require live browser testing.

### 5. FK Constraint on Category Delete

**Test:** Create an expense linked to "Herramientas". Then navigate to /catalogos > "Categorias de Gasto" and attempt to delete "Herramientas".
**Expected:** Delete fails with an error (toast or modal showing error message). The category is NOT removed.
**Why human:** Error path depends on DB FK rejection and frontend error toast handling of non-2xx response.

### 6. Sidebar Visual Order

**Test:** Open the app sidebar and observe the navigation order.
**Expected:** Groups in order — Inicio (no label) → Finanzas (Gastos) → Productos (no label) → Datos base (Catalogos, Proveedores, Insumos).
**Why human:** Visual layout of sidebar groups requires browser rendering.

---

## Commit Verification

| Commit | Hash | Subsystem |
|--------|------|-----------|
| feat(07-01): ExpenseCategory entity + CatalogsModule integration | `52107ec` | backend |
| feat(07-01): Expense entity + ExpensesModule + migration + AppModule | `3d4f2d5` | backend |
| feat(07-02): expenses frontend — page, components, sidebar | `358b2c7` | frontend |

All 3 commits verified present in respective sub-repo histories.

---

## Summary

Phase 7 automated checks pass at 100% (19/19 truths, all artifacts, all key links). The backend implementation is complete and correct: `ExpensesModule` with QueryBuilder filtering, `ExpenseCategory` integrated into the generic `CatalogsModule` dimension map (zero code duplication), and the migration with FK constraint and 6 seed categories. The frontend is fully wired: server-side page fetches current-month data, `ExpenseTable` manages all state and filter re-fetching via `apiClientFetch`, month grouping is real (not stubbed), dialogs use react-hook-form + zod, and role-based rendering is correctly gated on `isAdmin`. Six items require human verification for visual/runtime behavior that cannot be confirmed statically.

---

_Verified: 2026-03-11_
_Verifier: Claude (gsd-verifier)_
