---
created: 2026-03-06T14:06:35.821Z
title: Logical size ordering and contextual sizes per product type
area: ui
files:
  - nemea-front/src/components/products/ProductBatchCreateDialog.tsx
  - nemea-back/src/catalogs/entities/product-size.entity.ts
---

## Problem

Sizes don't display in logical order (should be XS, S, M, L, XL, XXL not alphabetical). Additionally, different product types need different size systems — leather goods use clothing sizes but mousepads need dimension-based sizes (20x30, 30x30). Currently all sizes are shared across all product types with no ordering or context.

## Solution

Two improvements:
1. **Size ordering**: Add a `sort_order` column to product_sizes (or use sku_code for ordering). Display sizes sorted by this order in the batch creation wizard and anywhere sizes appear.
2. **Contextual sizes per product type** (bigger change): Associate size sets with product types so each type can have its own relevant sizes. This may require a junction table (product_type_sizes) or a grouping field. Needs design discussion before implementing.
