---
phase: 09-product-ux
plan: 01
subsystem: frontend/products
tags: [product-table, hierarchy, bom-group-editor, ux]
dependency_graph:
  requires: []
  provides: [product-hierarchy, name-level-bom-editor, bom-divergence-badge]
  affects: [ProductTable, ProductTypeGroup, BomGroupEditorDialog]
tech_stack:
  added: []
  patterns: [3-level-collapsible-hierarchy, cost-based-divergence-heuristic]
key_files:
  created:
    - nemea-front/src/components/products/ProductFinishGroup.tsx
    - nemea-front/src/components/products/ProductNameGroup.tsx
  modified:
    - nemea-front/src/components/products/ProductTypeGroup.tsx
    - nemea-front/src/components/products/BomGroupEditorDialog.tsx
decisions:
  - "SKU column in ProductFinishGroup is Link to /productos/[id] (relocated from removed Nombre column)"
  - "BOM divergence detection: cost heuristic first, accurate BOM-based after dialog opens"
  - "onDivergenceDetected uses local divergent variable, not React state (batching safety)"
  - "colSpan explicitly 6 in ProductFinishGroup (6-column leaf table)"
metrics:
  duration: 6min
  completed: "2026-03-27"
  tasks_completed: 2
  tasks_total: 2
  files_created: 2
  files_modified: 2
---

# Phase 9 Plan 1: Product Table Hierarchy Summary

3-level collapsible product hierarchy (Type > Name > Finish) with BOM group editor rescoped to name-level and cost-based divergence badge.

## What Was Done

### Task 1: Create hierarchical grouping components (Type > Name > Finish)

Created two new components and refactored ProductTypeGroup to build the 3-level hierarchy:

- **ProductFinishGroup.tsx** (new): Innermost collapsible level. Renders a 6-column table (SKU, Color, Talle, Costo, Precio Venta, Margen). SKU column is a Link to `/productos/[id]` preserving product detail navigation from the removed Nombre column. Each row expands to ProductExpandedRow with colSpan=6.

- **ProductNameGroup.tsx** (new): Middle collapsible level. Sub-groups products by finish. Includes admin-only "Editar BOM grupal" and "Precio grupal" buttons. Shows "BOM personalizado" badge when cost-based heuristic detects divergence.

- **ProductTypeGroup.tsx** (refactored): Outermost level now delegates to ProductNameGroup. Removed inner Table, BomGroupEditorDialog, and expandedId state. Kept BatchPriceDialog at type level for broad pricing. Only "Precio grupal" button at type level (BOM editor moved to name level).

**Commit:** `2193aa2`

### Task 2: Wire BomGroupEditorDialog to name-level scope and add override badge

- Added optional `groupName` prop to BomGroupEditorDialog -- shows in dialog title as "Editar BOM grupal -- {groupName}".
- Added optional `onDivergenceDetected` callback. Called after `fetchAllBoms` using the local `divergent` Set variable (not the `divergentIds` state) to avoid React batching issues.
- ProductNameGroup passes both props. The "BOM personalizado" badge appears from cost heuristic initially and updates to accurate BOM-based detection after dialog opens.

**Commit:** `6638b0e`

## Deviations from Plan

None -- plan executed exactly as written.

## Verification Results

1. **Type safety**: `tsc --noEmit` passes with zero errors
2. **Hierarchy structure**: ProductTypeGroup > ProductNameGroup > ProductFinishGroup > product rows
3. **Column reduction**: 6 columns in leaf table (SKU, Color, Talle, Costo, Precio Venta, Margen)
4. **Product detail link**: SKU column is Link to /productos/[id] in ProductFinishGroup
5. **BOM group editor scope**: Only in ProductNameGroup (removed from ProductTypeGroup)
6. **Override indicator**: "BOM personalizado" badge from cost heuristic + dialog callback
7. **No regression**: ProductExpandedRow renders with colSpan=6
8. **Batch price at type level**: BatchPriceDialog in ProductTypeGroup
9. **Batch price at name level**: BatchPriceDialog in ProductNameGroup

## Self-Check: PASSED

All files verified present. All commits verified in git log.
