---
status: complete
phase: 09-product-ux
source: [09-01-SUMMARY.md]
started: 2026-03-28T00:00:00.000Z
updated: 2026-03-28T00:00:00.000Z
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

number: 1
name: Hierarchical product tree
expected: |
  Navigate to /productos. Products should display in a 3-level collapsible tree:
  Type (Billetera, Cinturon...) > Name (Hefesto, Artemisa...) > Finish (Lisa, Grabada...)
  Each level should be collapsible. Product rows appear under the finish level.
awaiting: user response

## Tests

### 1. Hierarchical product tree
expected: Products display in 3-level collapsible tree: Type > Name > Finish. Each level collapses independently.
result: pass

### 2. SKU links to product detail
expected: Click a SKU code (e.g., 1.1.1.7.0) in the product table. It should navigate to /productos/[id] product detail page.
result: pass

### 3. BOM group editor at name level
expected: Find the "Editar BOM grupal" button — it should appear next to a product NAME header (e.g., "Hefesto"), NOT at the type level (e.g., "Billetera"). Click it — the dialog title should say "Editar BOM grupal — Hefesto".
result: pass

### 4. Batch price at both levels
expected: "Precio grupal" button appears at BOTH the type level (e.g., next to "Cinturon") AND the name level (e.g., next to "Artemisa"). Both open BatchPriceDialog with their respective scoped products.
result: pass

### 5. BOM override indicator
expected: If products within a name group have different costs (different BOMs), a "BOM personalizado" badge should appear on the name header. If all have the same cost, no badge.
result: pass

### 6. Column layout correct
expected: The leaf table (under each finish) shows 6 columns: SKU, Color, Talle, Costo, Precio Venta, Margen. NOT 8 columns (Nombre and Terminación are now hierarchy headers, not columns).
result: pass

## Summary

total: 6
passed: 6
issues: 0
pending: 0
skipped: 0

## Gaps

[none yet]
