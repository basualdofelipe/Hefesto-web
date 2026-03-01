---
phase: 03-catalogs-and-suppliers
plan: 04
subsystem: ui
tags: [zod, react-hook-form, validation, gap-closure]

# Dependency graph
requires:
  - phase: 03-catalogs-and-suppliers/03-03
    provides: Supplier form, catalog tabs page
provides:
  - Supplier email field truly optional end-to-end (browser → zod → backend)
  - Empty string to undefined transformation for optional supplier fields
affects: [04-supplies]

# Tech tracking
tech-stack:
  added: []
  patterns: [empty-string-to-undefined-transform]

key-files:
  modified:
    - nemea-front/src/components/suppliers/SupplierForm.tsx
    - nemea-front/src/app/(app)/proveedores/nuevo/page.tsx
    - nemea-front/src/app/(app)/proveedores/[id]/editar/EditSupplierClient.tsx
    - nemea-front/src/app/(app)/catalogos/page.tsx
---

## Summary

Gap closure plan fixing 3 verification issues from Phase 3.

**Gap 1 (BLOCKER — FIXED): Supplier email validation chain broken.**
Three-layer fix: removed `type='email'` from HTML Input (no browser-native validation), kept zod `.email().or(z.literal('')).optional()` schema for client-side validation, and added `cleanSupplierData()` helper in both create and edit pages that transforms empty strings to `undefined` before API submission. Backend `@IsOptional()` now correctly skips `@IsEmail()` for undefined values.

**Gap 2 (BLOCKER — REVERTED): Catalog tabs mobile wrapping.**
Initial fix changed `flex-wrap` to `overflow-x-auto` with hidden scrollbar. Reverted after user testing revealed `justify-center` in TabsList base styles hid left-side tabs and fixed `h-9` caused vertical scroll bleed. Tabs remain `flex-wrap` (wraps on mobile) — functional but not ideal. Will be redesigned in a future UI polish pass.

**Gap 3 (CONFIRMED): Mobile sidebar backdrop.**
No code change needed. User confirmed sidebar backdrop renders and dismisses correctly on mobile viewports.

## Commits

| # | Hash | Message |
|---|------|---------|
| 1 | 894a9e6 | fix(03-04): fix supplier email validation and empty string handling |
| 2 | b301e40 | fix(03-04): replace catalog tabs flex-wrap with horizontal scroll on mobile |
| 3 | 3c56580 | revert(03-04): restore catalog tabs flex-wrap over horizontal scroll |

## Deviations

- **Tab scroll reverted**: The horizontal scroll approach had fundamental UX issues with the Shadcn TabsList component (justify-center hiding left tabs). Reverted to flex-wrap. The tab layout is temporary UI and will be redesigned later.

## Self-Check: PASSED

- [x] Gap 1 fixed: supplier form submits with blank email (verified by user)
- [x] Gap 2 reverted to working state (flex-wrap shows all tabs)
- [x] Gap 3 confirmed working by user
- [x] All commits made atomically
- [x] No regressions in existing functionality
