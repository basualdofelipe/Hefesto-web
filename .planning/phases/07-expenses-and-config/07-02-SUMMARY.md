---
phase: 07-expenses-and-config
plan: 02
subsystem: frontend
tags: [nextjs, react, expenses, crud, filters, sidebar]

requires:
  - phase: 07-expenses-and-config
    plan: 01
    provides: Expense and ExpenseCategory backend API
provides:
  - Expense list page at /finanzas/gastos with month grouping, filters, summary bar
  - Expense categories tab on /catalogos with inline CRUD
  - Sidebar restructured with Finanzas group
affects: []

tech-stack:
  added: []
  patterns: [Month grouping via Map, date string splitting to avoid timezone shift, getCurrentMonthRange for server-side default fetch]

key-files:
  created:
    - nemea-front/src/components/expenses/types.ts
    - nemea-front/src/components/expenses/ExpenseTable.tsx
    - nemea-front/src/components/expenses/ExpenseMonthGroup.tsx
    - nemea-front/src/components/expenses/ExpenseFormDialog.tsx
    - nemea-front/src/components/expenses/DeleteExpenseDialog.tsx
    - nemea-front/src/app/(app)/finanzas/gastos/page.tsx
  modified:
    - nemea-front/src/app/(app)/catalogos/page.tsx
    - nemea-front/src/components/layout/AppSidebar.tsx

key-decisions:
  - "Date display uses string splitting (not new Date()) to avoid timezone shift — '2026-03-06' -> '06/03/2026'"
  - "Summary bar totals derived via useMemo from current expenses array (not separate fetch)"
  - "Sidebar restructured: TOP_ITEMS > FINANZAS_ITEMS > PRODUCTOS_ITEMS > DATOS_BASE_ITEMS as separate arrays"
  - "getCategoryColor with lowercase lookup + gray fallback for user-created categories"
  - "All month groups open by default (defaultOpen=true on Collapsible)"

patterns-established:
  - "groupByMonth: Map<string, Expense[]> keyed by YYYY-MM substring"
  - "getCurrentMonthRange: compute first/last day of current month for default server-side fetch"
  - "formatDateDisplay: string split without Date constructor (timezone-safe)"

requirements-completed: [EXPN-01, EXPN-02, EXPN-03]

completed: 2026-03-11
---

# Phase 7 Plan 2: Expenses Frontend Summary

**Expense tracking UI: /finanzas/gastos with month grouping + filters + CRUD modals + expense categories on /catalogos + sidebar Finanzas group**

## Performance

- **Completed:** 2026-03-11
- **Tasks:** 2 (Task 1: implementation, Task 2: human verification — approved)
- **Files modified:** 8

## Accomplishments

- `/finanzas/gastos` page with expenses grouped by month, collapsible sections, colored category badges
- Summary bar showing total + count for filtered period, updates dynamically
- Category and date range filters working together, re-fetching from backend on change
- Create/edit expense modal (react-hook-form + zod, date defaults to today)
- Delete expense with AlertDialog confirmation (hard delete)
- Expense categories tab on `/catalogos` via DIMENSIONS array entry (zero extra code)
- Sidebar restructured: Inicio → Finanzas (Gastos) → Productos → Datos base
- USER role: view-only (no create/edit/delete buttons rendered)

## Task Commits

Implementation was done in a prior session (pre-SUMMARY). Human verification checkpoint approved 2026-03-11.

## Files Created/Modified

- `nemea-front/src/components/expenses/types.ts` — Expense/ExpenseCategory interfaces, CATEGORY_COLORS, getCategoryColor, formatAmount, formatMonthLabel, formatDateDisplay, groupByMonth, getCurrentMonthRange
- `nemea-front/src/components/expenses/ExpenseTable.tsx` — Client component: filters + month groups + summary bar + CRUD state
- `nemea-front/src/components/expenses/ExpenseMonthGroup.tsx` — Collapsible month group with expense rows and action buttons
- `nemea-front/src/components/expenses/ExpenseFormDialog.tsx` — Create/edit modal with react-hook-form + zod
- `nemea-front/src/components/expenses/DeleteExpenseDialog.tsx` — AlertDialog confirmation for hard delete
- `nemea-front/src/app/(app)/finanzas/gastos/page.tsx` — Server component: fetches categories + current-month expenses, passes to ExpenseTable
- `nemea-front/src/app/(app)/catalogos/page.tsx` — Added expense-categories entry to DIMENSIONS array
- `nemea-front/src/components/layout/AppSidebar.tsx` — Restructured into 4 separate nav arrays with Finanzas group

## Decisions Made

- Date display uses string splitting instead of `new Date()` to avoid timezone shift (PostgreSQL DATE returns "YYYY-MM-DD")
- Summary bar totals computed via useMemo from expenses array — no separate calculation or API call
- Sidebar uses 4 separate NavItem arrays (TOP, FINANZAS, PRODUCTOS, DATOS_BASE) rendered as independent SidebarGroups
- Color fallback for user-created categories defaults to gray (bg-gray-100 text-gray-800)

## Deviations from Plan

None — all must-haves implemented as specified.

## Issues Encountered

None.

## User Setup Required

None.

## Next Phase Readiness

Phase 7 complete. All v1 features implemented:
- Auth, Catalogs, Suppliers, Supplies + Price History, Products + BOM, Cost Calculation, Expenses

---
*Phase: 07-expenses-and-config*
*Completed: 2026-03-11*
