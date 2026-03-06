---
phase: quick
plan: 1
subsystem: frontend-products, backend-catalogs
tags: [bugfix, hydration, sku, catalogs]
dependency_graph:
  requires: []
  provides: [hydration-fix, catalog-sku-auto-assign]
  affects: [products-page, catalog-crud]
tech_stack:
  added: []
  patterns: [asChild-radix-pattern, max-plus-one-auto-increment]
key_files:
  created: []
  modified:
    - nemea-front/src/components/products/ProductTypeGroup.tsx
    - nemea-back/src/catalogs/catalogs.service.ts
decisions:
  - "Use asChild + div[role=button] instead of removing buttons from CollapsibleTrigger"
  - "DIMENSIONS_WITH_SKU set pattern for selective skuCode auto-assignment"
  - "MAX + 1 pattern with COALESCE for sequential skuCode generation"
metrics:
  duration: 2min
  completed: "2026-03-06T14:13:34Z"
---

# Quick Task 1: Fix Button Nesting Hydration Error and Catalog Creation Summary

**One-liner:** Fixed CollapsibleTrigger button nesting causing hydration errors and added auto-assign skuCode for product dimension catalog CRUD.

## What Was Done

### Task 1: Fix button nesting in ProductTypeGroup CollapsibleTrigger

- Added `asChild` prop to `CollapsibleTrigger` so it delegates rendering to a child `<div>` instead of its own `<button>`
- The div gets `role="button"`, `tabIndex={0}`, and keyboard handler (`Enter`/`Space`) for accessibility
- Buttons for "Editar BOM grupal" and "Precio grupal" remain inside without violating HTML nesting rules
- **Commit:** `953ee0c` (nemea-front)

### Task 2: Auto-assign skuCode in CatalogsService.create()

- Added `DIMENSIONS_WITH_SKU` readonly set identifying the 5 product dimensions that have `skuCode`
- Modified `create()` to query `MAX(skuCode) + 1` before insert when dimension needs skuCode and none provided
- `COALESCE(MAX, 0)` handles empty tables (first item gets `skuCode=1`)
- `supply-types` dimension (no skuCode column) completely unaffected
- **Commit:** `4497cfc` (nemea-back)

## Deviations from Plan

None - plan executed exactly as written.

## Verification

- Frontend build: passes clean (no errors or warnings related to this change)
- Backend build: passes clean (0 issues, 80 files compiled)

## Self-Check

- [x] nemea-front/src/components/products/ProductTypeGroup.tsx modified
- [x] nemea-back/src/catalogs/catalogs.service.ts modified
- [x] Commit 953ee0c exists in nemea-front
- [x] Commit 4497cfc exists in nemea-back
- [x] Both projects build successfully

## Self-Check: PASSED
