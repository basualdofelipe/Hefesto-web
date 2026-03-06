---
created: 2026-03-06T14:06:35.821Z
title: Table column alignment and visual polish
area: ui
files:
  - nemea-front/src/components/products/ProductTypeGroup.tsx
  - nemea-front/src/components/supplies/SupplyTypeGroup.tsx
---

## Problem

Tables (insumos, productos) have columns that don't align on the same line consistently. The visual layout feels messy — columns should be in a single row with proper spacing. This affects both the supplies and products grouped tables.

## Solution

UI polish pass on grouped tables:
- Ensure all column headers and data cells align in a single row
- Review column width distribution (fixed vs flex)
- Check responsive behavior
- Apply consistent spacing across all grouped table components
- Can be addressed in a dedicated UI polish phase
