---
status: complete
phase: 08-hardening
source: [08-01-SUMMARY.md, 08-02-SUMMARY.md]
started: 2026-03-28T00:00:00.000Z
updated: 2026-03-28T00:00:00.000Z
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

number: 1
name: Unauthenticated redirect
expected: |
  Open an incognito window (no session). Navigate to /productos directly. You should be redirected to /login without seeing any product data.
awaiting: user response

## Tests

### 1. Unauthenticated redirect
expected: Navigating to any app route without being logged in redirects to /login
result: pass

### 2. JWT expiry handling
expected: When JWT expires mid-session, the next API call redirects to /login instead of showing a broken UI or generic error
result: skipped
reason: "JWT TTL is 7 days — cannot force expiry in manual testing without modifying token"

### 3. USER role cannot access admin routes
expected: A USER-role account cannot access /productos, /insumos, /catalogos, /proveedores, /finanzas, /usuarios — gets redirected to /
result: issue
reported: "Routes are correctly blocked but sidebar still shows all links to USER role. Investor sees Productos, Catalogos, Proveedores, etc. but can't access any of them — confusing UX. Sidebar should hide admin-only links for USER role."
severity: major

### 4. Users admin page
expected: Admin can view /usuarios page listing all users with email, role, and active status. Can create a new user and toggle active/inactive.
result: pass

### 5. Acceso denegado page
expected: A user not in the whitelist sees a clear "Tu cuenta no está autorizada" message with admin contact info
result: pass
note: Acceso denegado works correctly on login attempt with disabled user. However, if admin disables a user mid-session, the next server-side apiFetch throws "API error: 401" instead of redirecting to /acceso-denegado. Minor edge case — apiFetch (server-side) lacks 401 handling unlike apiClientFetch (client-side).

### 6. DRY cleanup verified
expected: Shared types work correctly — products page loads (uses SupplyOption), supplies page loads (uses UNIT_LABELS), expense form category select works (watch fix)
result: pass
note: UI polish needed for both productos and insumos pages (future milestone). User wants insumos possibly grouped by proveedor instead of tipo.

### 7. Producción externa supply type
expected: In /catalogos, the supply types tab shows "Produccion externa" as an option
result: pass
note: Seeded via migration 1772700000000-SeedProduccionExterna.ts (ON CONFLICT DO NOTHING)

## Summary

total: 7
passed: 5
issues: 1
pending: 0
skipped: 1

## Gaps

- truth: "USER role should not see admin-only links in the sidebar"
  status: failed
  reason: "User reported: sidebar shows all links (Productos, Catalogos, Proveedores, etc.) to USER role but they can't access any. Confusing UX — should hide admin-only links for USER role."
  severity: major
  test: 3
  artifacts:
    - nemea-front/src/components/layout/AppSidebar.tsx
  missing:
    - "Sidebar should conditionally render links based on user role. Admin sees all, USER sees only Inicio, Calculadora, and Dashboard (when it exists)."
