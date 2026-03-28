---
created: 2026-03-26T00:00:00.000Z
title: Fix ExpenseFormDialog category Select not reflecting visual changes
area: bugfix
files:
  - nemea-front/src/components/expenses/ExpenseFormDialog.tsx
---

## Problem

The `<Select>` for category uses `value={isEdit ? expense.category.id : undefined}` which is static. When the user changes the category, the trigger doesn't update visually even though react-hook-form captures the change internally.

## Solution

Use `watch('categoryId')` as the `value` prop, same pattern used in ProductEditDialog. One-line fix.
