---
created: 2026-03-26T00:00:00.000Z
title: Delete deactivateBySupplier dead code in SuppliesService
area: cleanup
files:
  - nemea-back/src/supplies/supplies.service.ts
---

## Problem

`deactivateBySupplier()` method at supplies.service.ts:275-282 is never called. The cascade deactivation logic lives directly in `SuppliersService.toggleStatus()` via `this.supplyRepo.update()`. Dead code that confuses future readers.

## Solution

Delete the method. The logic already works correctly in SuppliersService.
