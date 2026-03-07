---
phase: 07-expenses-and-config
plan: 01
subsystem: api
tags: [nestjs, typeorm, expenses, crud, querybuilder, migration]

requires:
  - phase: 03-catalogs-and-suppliers
    provides: CatalogsModule dimension map pattern for generic CRUD
provides:
  - ExpenseCategory entity integrated into CatalogsModule dimension map
  - Expense entity with CRUD endpoints and date/category filtering
  - Migration with expense_categories and expenses tables + 6 seed categories
affects: [07-02-frontend-expenses]

tech-stack:
  added: []
  patterns: [QueryBuilder filtering with optional andWhere clauses, DATE type for calendar dates]

key-files:
  created:
    - nemea-back/src/catalogs/entities/expense-category.entity.ts
    - nemea-back/src/expenses/entities/expense.entity.ts
    - nemea-back/src/expenses/dto/create-expense.dto.ts
    - nemea-back/src/expenses/dto/update-expense.dto.ts
    - nemea-back/src/expenses/dto/query-expenses.dto.ts
    - nemea-back/src/expenses/expenses.service.ts
    - nemea-back/src/expenses/expenses.controller.ts
    - nemea-back/src/expenses/expenses.module.ts
    - nemea-back/src/database/migrations/1772600000000-CreateExpenseCategoriesAndExpenses.ts
  modified:
    - nemea-back/src/catalogs/catalogs.service.ts
    - nemea-back/src/catalogs/catalogs.controller.ts
    - nemea-back/src/catalogs/catalogs.module.ts
    - nemea-back/src/app.module.ts

key-decisions:
  - "Expense.amount typed as string (TypeORM decimal behavior, same as SupplyPriceHistory.price)"
  - "Expense.date uses PostgreSQL DATE type (not timestamptz) to avoid timezone shift on calendar dates"
  - "Hard delete on expenses (no soft delete) per user decision"
  - "ExpenseCategory integrated into CatalogsModule dimension map (zero new controller/service code for categories)"

patterns-established:
  - "QueryBuilder filtering: optional andWhere clauses for categoryId, dateFrom, dateTo"
  - "DATE column type for calendar dates to avoid timezone issues"

requirements-completed: [EXPN-01, EXPN-02, EXPN-03]

duration: 4min
completed: 2026-03-06
---

# Phase 7 Plan 1: Expenses Backend Summary

**ExpenseCategory as CatalogsModule dimension + Expense CRUD with QueryBuilder date/category filtering and migration with 6 seed categories**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-07T01:13:48Z
- **Completed:** 2026-03-07T01:17:30Z
- **Tasks:** 2
- **Files modified:** 13

## Accomplishments
- ExpenseCategory entity integrated into existing CatalogsModule dimension map with zero new controller/service code
- Full Expense CRUD with QueryBuilder filtering by categoryId, dateFrom, dateTo
- Migration creating both tables with 6 seed categories and date DESC index

## Task Commits

Each task was committed atomically:

1. **Task 1: ExpenseCategory entity + integrate into CatalogsModule dimension map** - `52107ec` (feat)
2. **Task 2: Expense entity + ExpensesModule (CRUD + filtering) + migration + AppModule registration** - `3d4f2d5` (feat)

## Files Created/Modified
- `nemea-back/src/catalogs/entities/expense-category.entity.ts` - ExpenseCategory entity extending BaseEntity
- `nemea-back/src/catalogs/catalogs.service.ts` - Added expense-categories to VALID_DIMENSIONS and dimensionMap
- `nemea-back/src/catalogs/catalogs.controller.ts` - Added expense-categories to DIMENSION_ENUM
- `nemea-back/src/catalogs/catalogs.module.ts` - Registered ExpenseCategory in TypeOrmModule.forFeature
- `nemea-back/src/expenses/entities/expense.entity.ts` - Expense entity with amount, concept, date, category FK
- `nemea-back/src/expenses/dto/create-expense.dto.ts` - DTO with validation for creating expenses
- `nemea-back/src/expenses/dto/update-expense.dto.ts` - PartialType of CreateExpenseDto
- `nemea-back/src/expenses/dto/query-expenses.dto.ts` - Optional filters: categoryId, dateFrom, dateTo
- `nemea-back/src/expenses/expenses.service.ts` - CRUD with QueryBuilder filtering
- `nemea-back/src/expenses/expenses.controller.ts` - REST endpoints with ADMIN role guards
- `nemea-back/src/expenses/expenses.module.ts` - Module importing Expense and ExpenseCategory repos
- `nemea-back/src/app.module.ts` - Registered ExpensesModule
- `nemea-back/src/database/migrations/1772600000000-CreateExpenseCategoriesAndExpenses.ts` - Tables + seed + index

## Decisions Made
- Expense.amount typed as string (TypeORM decimal returns string, consistent with SupplyPriceHistory.price pattern)
- Expense.date uses PostgreSQL DATE type (not timestamptz) to avoid timezone shift on calendar dates per plan's Pitfall 2 reference
- Hard delete on expenses (no is_active/soft delete) per user decision documented in plan
- ExpenseCategory integrated via CatalogsModule dimension map pattern -- reuses existing generic CRUD with zero duplication

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Expense backend API complete, ready for frontend implementation (plan 07-02)
- Categories accessible via existing /api/catalogs/expense-categories endpoints
- Expenses accessible via /api/expenses with filtering support

---
*Phase: 07-expenses-and-config*
*Completed: 2026-03-06*
