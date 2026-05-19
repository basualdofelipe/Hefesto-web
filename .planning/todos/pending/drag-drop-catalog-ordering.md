---
title: Drag & drop ordering for all catalog dimensions
priority: medium
area: ui
source: Phase 12.2 verification feedback
created: 2026-04-10
---

## Context

Phase 12.2 added a `sort_order` column to `product_sizes` only, with hardcoded values in a migration. The user expected to control the order from the UI, not have it hardcoded. The real need is: admin can reorder ANY catalog dimension (tipos, nombres, terminaciones, colores, talles) via drag & drop in the /catalogos page.

## What exists today

- `product_sizes` table has `sort_order SMALLINT NOT NULL DEFAULT 0` (added in Phase 12.2)
- `CatalogsService.findAll` already has a special case for `product-sizes` that orders by `sortOrder ASC`
- Other catalog tables (`product_types`, `product_names`, `product_finishes`, `product_colors`, `supply_types`) do NOT have `sort_order` — they order by `name ASC`
- All catalog entities extend `BaseEntity` (UUID PK + timestamps)

## What needs to happen

### Backend
1. Add `sort_order SMALLINT NOT NULL DEFAULT 0` column to ALL catalog tables: `product_types`, `product_names`, `product_finishes`, `product_colors`, `supply_types` (migration)
2. Add `sortOrder` property to each catalog entity (or to a shared base if possible — but `CatalogEntity` is a type alias, not a class)
3. Update `CatalogsService.findAll` to order ALL dimensions by `sortOrder ASC, name ASC` (remove the `product-sizes` special case — it becomes the general rule)
4. New endpoint: `PATCH /api/catalogs/:dimension/reorder` — accepts `{ items: [{ id: string, sortOrder: number }] }` and bulk-updates sort_order for the given dimension. Protected by `@RequirePermission('can_edit_products')` for product dimensions, `can_edit_supplies` for supply_types

### Frontend
1. Install `@dnd-kit/core` + `@dnd-kit/sortable` (or evaluate if Shadcn has a built-in sortable pattern)
2. In `/catalogos` page, each catalog tab gets drag handles on rows
3. Drag & drop reorders visually and calls the reorder endpoint on drop
4. The order persists across page reloads (comes from DB)
5. New items created via the inline CRUD default to `sort_order = 0` (appear at top until admin reorders)

### UX considerations
- Drag handle icon (GripVertical from lucide-react) on the left of each row
- Optimistic update on drop (instant visual feedback, API call in background)
- If API fails, revert to previous order + toast.error
- Mobile: drag & drop may not work well on touch — consider up/down arrow buttons as fallback or ignore mobile for now (admin-only feature, likely used on desktop)

## Scope note

This is a new capability (drag & drop reordering) — NOT a fix to the existing sort_order implementation. Phase 12.2's sort_order for sizes was the seed infrastructure; this todo extends it to all dimensions and adds the UI.
