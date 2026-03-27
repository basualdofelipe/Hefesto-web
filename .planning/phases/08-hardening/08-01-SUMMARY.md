---
phase: 08-hardening
plan: 01
subsystem: auth-middleware-backend
tags: [security, middleware, auth, cleanup, migration]
dependency_graph:
  requires: []
  provides: [middleware-auth, 401-redirect, 403-handling, toggle-status-endpoint, produccion-externa-seed]
  affects: [nemea-front/src/middleware.ts, nemea-front/src/lib/api-client.ts, nemea-back/src/users/users.controller.ts]
tech_stack:
  added: []
  patterns: [role-based-routing, soft-delete-toggle, middleware-auth-callback]
key_files:
  created:
    - nemea-front/src/middleware.ts
    - nemea-back/src/database/migrations/1772700000000-SeedProduccionExterna.ts
  modified:
    - nemea-front/src/auth.ts
    - nemea-front/src/lib/api-client.ts
    - nemea-front/src/app/(auth)/acceso-denegado/page.tsx
    - nemea-back/src/users/users.controller.ts
    - nemea-back/src/users/users.controller.spec.ts
    - nemea-back/src/supplies/supplies.service.ts
  deleted:
    - nemea-front/src/proxy.ts
decisions:
  - "proxy.ts renamed to middleware.ts with export name 'middleware' for Next.js 16 compatibility"
  - "UsersController hard DELETE replaced with PATCH toggle-status (soft delete pattern)"
  - "deactivateBySupplier dead code removed from SuppliesService"
metrics:
  duration: 5min
  completed: "2026-03-27"
  tasks_completed: 3
  tasks_total: 3
---

# Phase 8 Plan 1: Auth & Middleware Hardening Summary

Next.js middleware wired correctly (proxy.ts renamed to middleware.ts with role-based routing), apiClientFetch handles 401/403 distinctly, UsersController uses soft-delete toggle-status, and 'Produccion externa' supply type seeded.

## What Was Done

### Task 1: Fix middleware + 401/403 handling + role-based routing
- Created `nemea-front/src/middleware.ts` replacing broken `proxy.ts` (export name was `proxy`, not `middleware` -- Next.js 16 was not intercepting any routes)
- Middleware now handles: unauthenticated redirect to `/login`, no-backend-token redirect to `/acceso-denegado`, role-based admin-only route protection (USER role redirected to `/`)
- Admin-only routes: `/catalogos`, `/proveedores`, `/insumos`, `/productos`, `/finanzas`, `/usuarios`
- Updated `apiClientFetch` to handle 401 (redirect to `/login` on JWT expiry) and 403 (descriptive permission error) before generic error handling
- Updated `auth.ts` comment from "proxy.ts" to "middleware.ts" reference
- Improved `acceso-denegado` page with investor-friendly message including admin contact email

### Task 2: Backend user toggle-status + dead code cleanup + produccion externa seed
- Replaced `DELETE /users/:id` with `PATCH /users/:id/toggle-status` endpoint using proper null checks (no unsafe casts)
- Removed `deactivateBySupplier` dead code from `SuppliesService` (zero references outside its own definition)
- Created idempotent seed migration for 'Produccion externa' supply type (`ON CONFLICT DO NOTHING`)
- Updated controller tests to match new toggle-status behavior (3 test cases: deactivate, activate, not found)

### Task 3: Verify role enforcement on all existing controllers
- Confirmed 25 `@Roles(Role.ADMIN)` decorators across all 6 controllers
- All mutation endpoints (POST, PATCH, PUT, DELETE) are role-protected
- All GET endpoints are accessible to any authenticated user (protected by global JwtAuthGuard only)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Updated users.controller.spec.ts for toggle-status**
- **Found during:** Task 2
- **Issue:** Test file referenced the deleted `remove()` method, causing TypeScript compilation error
- **Fix:** Replaced DELETE test with 3 toggle-status tests (deactivate active user, activate inactive user, NotFoundException for missing user)
- **Files modified:** `nemea-back/src/users/users.controller.spec.ts`
- **Commit:** c56509d

## Verification Results

| Check | Result |
|-------|--------|
| Frontend `tsc --noEmit` | PASS (zero errors) |
| Backend `tsc --noEmit` | PASS (zero errors) |
| middleware.ts export exists | PASS |
| 401 handling in api-client | PASS |
| toggle-status endpoint | PASS |
| deactivateBySupplier deleted | PASS (zero matches) |
| ProduccionExterna migration exists | PASS |
| proxy.ts deleted | PASS |
| auth.ts comment updated | PASS |

## Commits

| Task | Commit | Repo | Message |
|------|--------|------|---------|
| 1 | af89f02 | nemea-front | feat(08-01): fix middleware + 401/403 handling + role-based routing |
| 2 | c56509d | nemea-back | feat(08-01): backend toggle-status + dead code cleanup + produccion externa seed |
| 3 | (verification only) | — | No code changes needed |

## Known Stubs

None -- all functionality is fully wired.

## Self-Check: PASSED

All created/modified files verified on disk. All commit hashes found in git log. proxy.ts confirmed deleted.
