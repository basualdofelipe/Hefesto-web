---
phase: 12-scenarios
plan: 03
subsystem: scenarios
tags: [gap-closure, uat-fix, admin-permissions, inactive-products]
dependency_graph:
  requires: [12-01, 12-02]
  provides: [admin-delete-shared-scenarios, inactive-products-in-editor]
  affects: [scenarios-controller, scenarios-service, scenario-card, scenario-list]
tech_stack:
  added: []
  patterns: [role-based-query-filter, canDelete-prop-pattern]
key_files:
  created: []
  modified:
    - nemea-back/src/scenarios/scenarios.service.ts
    - nemea-back/src/scenarios/scenarios.controller.ts
    - nemea-front/src/app/(app)/escenarios/[id]/page.tsx
    - nemea-front/src/components/scenarios/ScenarioCard.tsx
    - nemea-front/src/app/(app)/escenarios/ScenarioListClient.tsx
decisions:
  - "Admin delete bypasses owner filter via role param in remove(), not a separate admin-only endpoint"
  - "canDelete prop on ScenarioCard separates delete visibility from ownership for extensibility"
metrics:
  duration: 3min
  completed: 2026-03-29
---

# Phase 12 Plan 03: Gap Closure Summary

Patched 2 UAT gaps: inactive products now show in scenario editor, admin can delete any public scenario.

## What Was Done

### Task 1: Inactive products fetch + admin delete backend (backend@1a9d9ff, frontend@c67f01a)

**Gap 1 -- Inactive products in scenario editor:**
- Changed `/api/products` to `/api/products?includeInactive=true` in the scenario editor RSC page
- No other changes needed -- ProductOverrideTable already had badge + opacity styling for inactive products

**Gap 2 -- Admin delete backend:**
- Added `Role` import from `../common/types/role.enum` to `scenarios.service.ts`
- Updated `remove()` signature to accept `role: string` third parameter
- Admin role bypasses owner filter (`where: { id }` only), regular users keep existing `{ id, user: { id: userId } }` filter
- Updated controller to extract `@CurrentUser('role')` and pass to service
- Updated `@ApiOperation` summary from "owner only" to "owner or admin"

### Task 2: Admin delete button on shared scenario cards (frontend@f9e5010)

**ScenarioCard.tsx:**
- Added `canDelete: boolean` prop to `ScenarioCardProps` interface
- Changed delete button conditional from `{isOwner &&` to `{canDelete &&`
- Toggle-public and edit/view buttons remain gated by `isOwner` (unchanged)

**ScenarioListClient.tsx:**
- Imported `useIsAdmin` hook
- Added `const isAdmin = useIsAdmin()` declaration
- Own scenarios pass `canDelete={true}` (owner always can delete)
- Shared scenarios pass `canDelete={isAdmin}` (only admin sees delete button)
- Shared scenarios `onDelete` wired to `setDeleteTarget(scenario)` instead of `() => {}` so the confirmation dialog opens for admin

## Verification Results

- Backend: `npx nest build` -- 0 issues, 126 files compiled
- Frontend: `npx next build` -- success, all routes compiled
- Frontend lint: clean (no ESLint errors on modified files)

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all functionality is fully wired.

## Decisions Made

1. **Role-based query filter in remove()**: Admin bypasses owner filter via conditional `where` clause in the same method, rather than a separate admin-only endpoint. This keeps the API surface minimal (one DELETE endpoint handles both cases).
2. **canDelete prop pattern**: Separated delete visibility from ownership via a new boolean prop, making the component more flexible without changing the isOwner prop semantics for other UI elements (toggle-public, edit/view).

## Self-Check: PASSED

- All 5 modified files exist on disk
- backend@1a9d9ff: FOUND
- frontend@c67f01a: FOUND
- frontend@f9e5010: FOUND
- SUMMARY.md: FOUND
