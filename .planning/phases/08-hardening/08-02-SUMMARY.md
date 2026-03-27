---
phase: 08-hardening
plan: 02
subsystem: dry-cleanup-users-admin
tags: [refactor, dry, users, admin, sidebar, expense-fix]
dependency_graph:
  requires: [08-01]
  provides: [shared-supply-types, shared-formatters, shared-suppliers, supply-combobox, users-admin-page, admin-sidebar]
  affects: [nemea-front/src/types/supply.ts, nemea-front/src/lib/formatters.ts, nemea-front/src/lib/suppliers.ts, nemea-front/src/components/products/SupplyCombobox.tsx, nemea-front/src/app/(app)/usuarios/]
tech_stack:
  added: []
  patterns: [shared-types-module, shared-utility-module, extracted-component, watch-controlled-select]
key_files:
  created:
    - nemea-front/src/types/supply.ts
    - nemea-front/src/lib/formatters.ts
    - nemea-front/src/lib/suppliers.ts
    - nemea-front/src/components/products/SupplyCombobox.tsx
    - nemea-front/src/app/(app)/usuarios/page.tsx
    - nemea-front/src/app/(app)/usuarios/UsersClient.tsx
  modified:
    - nemea-front/src/components/supplies/types.ts
    - nemea-front/src/components/products/BomEditorDialog.tsx
    - nemea-front/src/components/products/BomGroupEditorDialog.tsx
    - nemea-front/src/components/products/ProductExpandedRow.tsx
    - nemea-front/src/components/products/ProductDetailClient.tsx
    - nemea-front/src/components/products/ProductTable.tsx
    - nemea-front/src/components/products/ProductTypeGroup.tsx
    - nemea-front/src/components/products/PriceHistoryDialog.tsx
    - nemea-front/src/components/supplies/PriceHistoryDialog.tsx
    - nemea-front/src/components/supplies/SupplyExpandedRow.tsx
    - nemea-front/src/app/(app)/productos/page.tsx
    - nemea-front/src/app/(app)/productos/[id]/page.tsx
    - nemea-front/src/app/(app)/proveedores/nuevo/page.tsx
    - nemea-front/src/app/(app)/proveedores/[id]/editar/EditSupplierClient.tsx
    - nemea-front/src/components/layout/AppSidebar.tsx
    - nemea-front/src/components/expenses/ExpenseFormDialog.tsx
decisions:
  - "SupplyOption includes optional supplier field for BOM editors backward compatibility"
  - "supplies/types.ts re-exports UNIT_LABELS via import+export (not export-from) to keep local usage working in formatPrice"
  - "formatAmount NOT extracted to avoid naming collision with expenses/types.ts version"
  - "Users admin prevents self-deactivation by comparing session.user.id"
metrics:
  duration: 9min
  completed: "2026-03-27"
  tasks_completed: 2
  tasks_total: 2
---

# Phase 8 Plan 2: DRY Cleanup + Users Admin Summary

Extracted 5 duplicated definitions (SupplyOption 8x, UNIT_LABELS 6x, formatDate 4x, cleanSupplierData 2x, SupplyCombobox 2x) into shared modules, built /usuarios admin page with CRUD, and added admin sidebar section.

## What Was Done

### Task 1: Extract shared types, formatters, and components (DRY cleanup)
- Created `src/types/supply.ts` with SupplyOption interface (includes optional supplier field) and UNIT_LABELS constant
- Created `src/lib/formatters.ts` with formatDate function including 2-digit day/month/year format options
- Created `src/lib/suppliers.ts` with cleanSupplierData using SupplierFormData type (not Record)
- Extracted SupplyCombobox from BomEditorDialog into `src/components/products/SupplyCombobox.tsx` without Check/cn unused imports
- Updated `supplies/types.ts` to import+re-export UNIT_LABELS from shared location
- Updated all 14 consumer files to import from shared locations
- Removed unused ChevronsUpDown, Command*, Popover* imports from BomEditorDialog and BomGroupEditorDialog
- Net reduction: 178 lines removed across the codebase

### Task 2: Users admin page + sidebar admin section + expense form fix
- Created `/usuarios` RSC page with server-side data fetching and admin-only redirect
- Created `UsersClient` with:
  - Users table showing email, name, role (badge), active status (badge), creation date
  - "Nuevo usuario" dialog with email (required), name (optional), role (Select: admin/user)
  - Toggle status button per row using PATCH /api/users/:id/toggle-status
  - Self-deactivation prevention (compares user.id with session.user.id)
  - Toast notifications for success/error, router.refresh() for data reload
- Added Admin sidebar group in AppSidebar (visible only to admin role via useSession)
- Fixed ExpenseFormDialog category Select: replaced static `expense.category.id` with `watch('categoryId')` for proper controlled component behavior

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed supplies/types.ts UNIT_LABELS re-export**
- **Found during:** Task 1
- **Issue:** Simple `export { UNIT_LABELS } from '@/types/supply'` syntax made UNIT_LABELS unavailable as a local variable, breaking formatPrice which references it
- **Fix:** Used `import { UNIT_LABELS } from '@/types/supply'; export { UNIT_LABELS };` pattern instead
- **Files modified:** `nemea-front/src/components/supplies/types.ts`
- **Commit:** 66d40bc

## Verification Results

| Check | Result |
|-------|--------|
| `tsc --noEmit` | PASS (zero errors) |
| SupplyOption interface count | 1 (was 8) |
| formatDate function count | 1 (was 4) |
| cleanSupplierData function count | 1 (was 2) |
| SupplyCombobox file exists | PASS |
| No unused imports in SupplyCombobox | PASS (0 matches for Check/utils) |
| formatDate includes 2-digit options | PASS |
| /usuarios page exists | PASS |
| Expense form watch fix | PASS |
| Sidebar admin section | PASS |

## Commits

| Task | Commit | Repo | Message |
|------|--------|------|---------|
| 1 | 66d40bc | nemea-front | refactor(08-02): extract shared types, formatters, and components (DRY cleanup) |
| 2 | 223dc04 | nemea-front | feat(08-02): users admin page + sidebar admin section + expense form fix |

## Known Stubs

None -- all functionality is fully wired. Users CRUD connects to backend endpoints created in 08-01. Shared modules are imported by all former duplication sites.

## Self-Check: PASSED

All created files verified on disk. All commit hashes found in git log.
