---
phase: 02-auth
verified: 2026-03-01T17:00:00Z
status: human_needed
score: 5/5 must-haves verified
human_verification:
  - test: "Complete Google OAuth flow end-to-end"
    expected: "User clicks 'Ingresar con Google', completes OAuth, arrives at / with persistent session; header shows avatar and dropdown"
    why_human: "Requires live Google credentials, browser interaction, and HTTP-only cookie inspection — cannot verify programmatically"
  - test: "Non-whitelisted user is redirected to /acceso-denegado"
    expected: "User authenticates with Google but their email is not in the users table; they see the branded access-denied page with 'Cerrar sesion' button"
    why_human: "Requires two Google accounts and live backend — cannot simulate in code analysis"
  - test: "Removing email from DB causes next API request to fail immediately"
    expected: "ADMIN removes a USER from the users table via DELETE /api/users/:id; the USER's next API call returns 401 without waiting for JWT expiry"
    why_human: "Requires live DB manipulation, active JWT session, and real API request — tests the whitelist-per-request revocation guarantee"
  - test: "callbackUrl redirect works after login"
    expected: "Visiting /some-protected-page redirects to /login?callbackUrl=/some-protected-page; after Google login, user lands on /some-protected-page"
    why_human: "Requires browser navigation flow testing — cannot verify from code alone"
---

# Phase 2: Auth Verification Report

**Phase Goal:** Users can log in with Google and the backend enforces authentication and roles on every request
**Verified:** 2026-03-01T17:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths (from Plan 02-01 Must-Haves)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User entity exists in DB with email, name, picture_url, google_id, role, is_active columns | VERIFIED | `nemea-back/src/users/entities/user.entity.ts` — all 6 columns present, extends BaseEntity |
| 2 | First ADMIN user is seeded via migration (reproducible across environments) | VERIFIED | `1772369233961-CreateUserTable.ts` — INSERT with email=basualdofelipe@gmail.com, role=admin |
| 3 | All backend routes return 401 when called without a valid JWT (global guard) | VERIFIED | `auth.module.ts` — JwtAuthGuard registered as APP_GUARD; E2E tests in `app.e2e-spec.ts` confirm /auth/me, /users all return 401 without JWT |
| 4 | Routes marked with @Public() skip authentication (health, swagger) | VERIFIED | `app.controller.ts` line 17 — @Public() on getHealth(); E2E test confirms GET /api/health returns 200 without JWT |
| 5 | Routes marked with @Roles(Role.ADMIN) return 403 for USER role tokens | VERIFIED | `roles.guard.ts` — throws ForbiddenException('Permisos insuficientes') when user.role not in requiredRoles; `users.controller.ts` — @Roles(Role.ADMIN) on all write endpoints |
| 6 | JWT strategy checks user whitelist on every request (not just at login) | VERIFIED | `jwt.strategy.ts` lines 29-31 — validate() calls usersService.findActiveByEmail(payload.email) on every request |

**Score:** 6/6 Plan 02-01 truths verified

### Observable Truths (from Plan 02-02 Must-Haves)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | POST /api/auth/google accepts Google id_token and returns backend JWT + user profile | VERIFIED | `auth.service.ts` — verifyGoogleIdToken + findActiveByEmail + jwtService.sign all wired; `auth.controller.ts` — @Public on POST /auth/google |
| 2 | POST /api/auth/google returns 401 for non-whitelisted emails | VERIFIED | `auth.service.ts` lines 31-34 — throws UnauthorizedException('Usuario no autorizado') when findActiveByEmail returns null |
| 3 | GET /api/auth/me returns authenticated user's profile from the JWT | VERIFIED | `auth.controller.ts` lines 47-65 — uses @CurrentUser() to get user, calls authService.getProfile(user.id) |
| 4 | POST /api/users (ADMIN only) adds email to whitelist | VERIFIED | `users.controller.ts` lines 38-51 — @Roles(Role.ADMIN), calls usersService.create(dto) with duplicate check |
| 5 | DELETE /api/users/:id (ADMIN only) removes user from whitelist | VERIFIED | `users.controller.ts` lines 54-61 — @Roles(Role.ADMIN), ParseIntPipe, calls usersService.remove(id) |
| 6 | USER role cannot access POST /api/users or DELETE /api/users/:id (403) | VERIFIED | RolesGuard throws ForbiddenException for non-ADMIN; confirmed by guard wiring in auth.module.ts |

**Score:** 6/6 Plan 02-02 truths verified

### Observable Truths (from Plan 02-03 Must-Haves)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can click 'Ingresar con Google' on /login and complete OAuth flow | HUMAN NEEDED | login page exists with Google button and server action; requires live Google credentials to verify |
| 2 | After login, user is redirected to original page (callbackUrl) | HUMAN NEEDED | proxy.ts line 9 adds callbackUrl; login/page.tsx line 48 passes redirectTo to signIn; requires browser testing |
| 3 | Session persists across browser restarts (HTTP-only cookie) | HUMAN NEEDED | NextAuth JWT strategy configured; cookie behavior requires browser inspection |
| 4 | Non-whitelisted user sees /acceso-denegado with friendly message | HUMAN NEEDED | proxy.ts lines 13-21 redirects users without accessToken; requires two Google accounts to test |
| 5 | Unauthenticated users are redirected to /login when accessing any protected page | VERIFIED | proxy.ts lines 6-11 — redirects !req.auth users to /login with callbackUrl |
| 6 | Session contains backendToken, role, email, name, and image | VERIFIED | auth.ts — jwt callback stores token.backendToken/role/userId; session callback maps to session.accessToken/user.role/user.id; next-auth.d.ts augments types |
| 7 | useIsAdmin() hook reads role from session and returns boolean | VERIFIED | `hooks/useIsAdmin.ts` — returns session?.user?.role === 'admin' |
| 8 | Authenticated pages show header with avatar dropdown and 'Cerrar sesion' | VERIFIED | `Header.tsx` — useSession, Avatar, DropdownMenu with signOut; wired in layout.tsx line 56 |

**Score:** 5/8 fully automated (3 require human verification — all auth flow dependent)

---

## Required Artifacts

### Plan 02-01 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/users/entities/user.entity.ts` | User entity extending BaseEntity | VERIFIED | All 6 columns present; extends BaseEntity; @Entity('users') @Unique(['email']) |
| `nemea-back/src/auth/guards/jwt-auth.guard.ts` | Global JWT guard with @Public() opt-out | VERIFIED | extends AuthGuard('jwt'); canActivate checks IS_PUBLIC_KEY; handleRequest throws 401 |
| `nemea-back/src/auth/guards/roles.guard.ts` | Role-based access control guard | VERIFIED | implements CanActivate; reads ROLES_KEY; throws ForbiddenException for insufficient roles |
| `nemea-back/src/auth/strategies/jwt.strategy.ts` | Passport JWT strategy with whitelist check | VERIFIED | ExtractJwt.fromAuthHeaderAsBearerToken(); validate() calls findActiveByEmail |
| `nemea-back/src/auth/auth.module.ts` | AuthModule with JwtModule, PassportModule, guards as APP_GUARD | VERIFIED | Both guards registered; JwtAuthGuard FIRST, RolesGuard SECOND |

### Plan 02-02 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/auth/auth.service.ts` | Google id_token verification + JWT minting | VERIFIED | OAuth2Client.verifyIdToken + findActiveByEmail + jwtService.sign all present |
| `nemea-back/src/auth/auth.controller.ts` | POST /auth/google and GET /auth/me | VERIFIED | @Public on POST; @CurrentUser on GET; both wired to authService methods |
| `nemea-back/src/users/users.controller.ts` | POST /users and DELETE /users/:id (ADMIN only) | VERIFIED | @Roles(Role.ADMIN) on all endpoints; ParseIntPipe on :id param |

### Plan 02-03 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-front/src/auth.ts` | NextAuth v5 config with Google provider, JWT/session callbacks | VERIFIED | Google provider; jwt callback calls backend; session callback maps tokens |
| `nemea-front/src/proxy.ts` | Route protection — redirects unauthenticated to /login | VERIFIED | Correctly uses auth() from @/auth; redirects !req.auth to /login; redirects no-backendToken to /acceso-denegado; config exports matcher |
| `nemea-front/src/app/login/page.tsx` | Branded login page with Google button | VERIFIED | Nemea engraving title; Google SVG icon; server action signIn with callbackUrl |
| `nemea-front/src/app/acceso-denegado/page.tsx` | Access denied page for non-whitelisted users | VERIFIED | Friendly message; signOut button; 'use client' for next-auth/react |
| `nemea-front/src/providers/SessionProvider.tsx` | Client wrapper for NextAuth SessionProvider | VERIFIED | 'use client'; wraps NextAuthSessionProvider |
| `nemea-front/src/lib/api.ts` | Fetch wrapper attaching Authorization: Bearer from session | VERIFIED | calls auth(); attaches Bearer header; throws on missing session or error status |
| `nemea-front/src/types/next-auth.d.ts` | TypeScript module augmentation for Session, JWT, User | VERIFIED | Session.accessToken; user.role; JWT.backendToken/role/userId |
| `nemea-front/src/components/layout/Header.tsx` | App header with user avatar dropdown and signOut | VERIFIED | useSession; Avatar; DropdownMenu; signOut with callbackUrl /login |
| `nemea-front/src/hooks/useIsAdmin.ts` | Client hook returning boolean for ADMIN role check | VERIFIED | useSession(); returns session?.user?.role === 'admin' |

---

## Key Link Verification

### Plan 02-01 Key Links

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `auth.module.ts` | APP_GUARD providers | JwtAuthGuard FIRST, RolesGuard SECOND | WIRED | Lines 29-37 in auth.module.ts — JwtAuthGuard registered before RolesGuard |
| `jwt.strategy.ts` | `users.service.ts` | findActiveByEmail() in validate() | WIRED | Line 29 — usersService.findActiveByEmail(payload.email) |
| `app.module.ts` | `auth.module.ts` | imports array | WIRED | Line 6 — AuthModule imported in AppModule |
| `app.controller.ts` | `public.decorator.ts` | @Public() on health endpoint | WIRED | Line 3 import + line 17 @Public() decorator |

### Plan 02-02 Key Links

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `auth.service.ts` | google-auth-library | OAuth2Client.verifyIdToken() | WIRED | Lines 69-71 — this.googleClient.verifyIdToken({idToken, audience}) |
| `auth.service.ts` | `users.service.ts` | findActiveByEmail for whitelist | WIRED | Line 31 — usersService.findActiveByEmail(payload.email) |
| `auth.service.ts` | @nestjs/jwt JwtService | jwtService.sign() | WIRED | Lines 43-47 — this.jwtService.sign({sub, email, role}) |
| `auth.controller.ts` | `auth.service.ts` | authService.validateGoogleToken() | WIRED | Line 44 — calls authService.validateGoogleToken(dto.idToken) |
| `users.controller.ts` | @Roles(Role.ADMIN) | Write endpoints restricted to ADMIN | WIRED | Lines 30, 39, 55 — @Roles(Role.ADMIN) on all endpoints |

### Plan 02-03 Key Links

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `auth.ts` | backend POST /api/auth/google | fetch in jwt callback when account present | WIRED | Lines 34-40 — fetch to /api/auth/google with idToken in jwt callback |
| `proxy.ts` | `auth.ts` | imports auth() for session check | WIRED | Line 1 — import { auth } from '@/auth' |
| `app/layout.tsx` | `SessionProvider.tsx` | wraps children with SessionProvider | WIRED | Line 7 import + line 55 — SessionProvider wrapping children |
| `lib/api.ts` | `auth.ts` | imports auth() to get session.accessToken | WIRED | Line 1 — import { auth } from '@/auth'; line 9 — const session = await auth() |
| `Header.tsx` | next-auth/react | useSession() + signOut() | WIRED | Line 5 — import { useSession, signOut } from 'next-auth/react' |
| `app/layout.tsx` | `Header.tsx` | Header rendered in authenticated layout | WIRED | Line 8 import + line 56 — Header rendered inside SessionProvider |

---

## Requirements Coverage

| Requirement | Source Plans | Description | Status | Evidence |
|-------------|--------------|-------------|--------|----------|
| AUTH-01 | 02-02, 02-03 | User can log in with Google OAuth and stay logged in across sessions | NEEDS HUMAN | auth.ts + login page + NextAuth session strategy configured; requires live OAuth flow to confirm session persistence |
| AUTH-02 | 02-01, 02-02, 02-03 | Only whitelisted emails can access the app | SATISFIED | JwtStrategy.validate() + AuthService.validateGoogleToken() both check findActiveByEmail; proxy.ts redirects if no backendToken |
| AUTH-03 | 02-01, 02-02 | ADMIN role can create, edit, and delete all resources | SATISFIED | RolesGuard wired; @Roles(Role.ADMIN) on all UsersController write endpoints; guards registered in correct order |
| AUTH-04 | 02-01, 02-03 | USER role can view but cannot modify | SATISFIED | RolesGuard allows through any authenticated user on read endpoints; useIsAdmin() hook in frontend for conditional UI |
| AUTH-05 | 02-01 | Backend validates JWT from NextAuth on every request and checks email whitelist | SATISFIED | JwtStrategy.validate() calls findActiveByEmail on every request; registered as global APP_GUARD via AuthModule |

All 5 requirements from Plans 02-01, 02-02, and 02-03 are accounted for:
- Plan 02-01 claims: AUTH-02, AUTH-03, AUTH-04, AUTH-05 — all verified
- Plan 02-02 claims: AUTH-01, AUTH-02, AUTH-05 — AUTH-01 needs human, AUTH-02/AUTH-05 verified
- Plan 02-03 claims: AUTH-01, AUTH-02, AUTH-04 — AUTH-01 needs human, AUTH-02/AUTH-04 verified

No orphaned requirements found in REQUIREMENTS.md for Phase 2. All 5 AUTH requirements are fully claimed and traceable.

---

## Anti-Patterns Scan

Files scanned from SUMMARY key-files sections across all three plans.

| File | Pattern | Severity | Finding |
|------|---------|----------|---------|
| `auth.service.ts` | Stub check | — | No stubs — full Google token verification + JWT minting implemented |
| `jwt.strategy.ts` | Whitelist only at login | — | No issue — validate() runs on every request, not just login |
| `auth.module.ts` | Guard registration order | — | No issue — JwtAuthGuard registered before RolesGuard |
| `proxy.ts` | Broad matcher | — | No issue — matcher correctly excludes api, _next/static, _next/image, favicon.ico, fonts, images |
| `auth.ts` | Double backend call (Pitfall 2) | — | No issue — backend called only in jwt callback, not in signIn callback |
| `auth.ts` | AUTH_SECRET vs JWT_SECRET confusion (Pitfall 5) | — | No issue — AUTH_SECRET for frontend, JWT_SECRET for backend, documented separately |
| `next-auth.d.ts` | TypeScript augmentation completeness | — | Session.accessToken, user.role, JWT.backendToken/role/userId all declared |

No blocker anti-patterns found.

---

## Commit Verification

| Commit | Branch | Description | Verified |
|--------|--------|-------------|---------|
| `95a7fc3` | nemea-back/development | User entity, migration, UsersModule, UsersService | YES |
| `649c07a` | nemea-back/development | Auth decorators, guards, JWT strategy, AuthModule, wiring | YES |
| `31233ae` | nemea-back/development | Failing tests for AuthService and AuthController (TDD RED) | YES |
| `2d65c82` | nemea-back/development | AuthService and AuthController implementation (TDD GREEN) | YES |
| `8e7f4ca` | nemea-back/development | Failing tests for UsersController and E2E guard behavior (TDD RED) | YES |
| `99aa7aa` | nemea-back/development | UsersController and E2E tests (TDD GREEN) | YES |
| `d3a7219` | nemea-front/development | NextAuth config, proxy, SessionProvider, api.ts | YES |
| `f6c366a` | nemea-front/development | Login page and acceso-denegado page | YES |
| `d9d5eca` | nemea-front/development | Header with avatar dropdown and useIsAdmin | YES |
| `ac64d1a` | nemea-back/development | Admin seed email updated to real user | YES |

All 10 commits verified in git log.

---

## Human Verification Required

### 1. Complete Google OAuth Flow

**Test:** Boot frontend (`npm run dev` in nemea-front/) and backend (`docker compose up -d && npm run start:dev` in nemea-back/). Visit `http://localhost:3000`. Click "Ingresar con Google". Sign in with the admin Google account (basualdofelipe@gmail.com).
**Expected:** OAuth flow completes, user is redirected to `/`, header shows NEMEA branding on the left and user avatar on the right. Clicking the avatar shows name, email, and "Cerrar sesion" option.
**Why human:** Requires live Google OAuth credentials, browser interaction, and real network calls to Google's auth servers.

### 2. Non-Whitelisted User Gets Access Denied

**Test:** Attempt to sign in with a Google account whose email is NOT in the users table.
**Expected:** OAuth completes with Google, but the user sees the `/acceso-denegado` page with "Tu cuenta no tiene acceso a Nemea. Contacta al administrador." and a "Cerrar sesion" button. No header is shown on this page.
**Why human:** Requires a second Google account not in the whitelist; cannot simulate without live credentials.

### 3. Instant Whitelist Revocation

**Test:** Sign in as a USER (not ADMIN). In another browser or Swagger, call DELETE /api/users/:id as ADMIN to remove the USER's record. Return to the USER session and make any API call.
**Expected:** The next API call returns 401 immediately, even though the JWT has not expired. This confirms whitelist is checked on every request in JwtStrategy.validate().
**Why human:** Requires two active sessions, live DB manipulation, and observation of 401 response.

### 4. callbackUrl Redirect

**Test:** While logged out, visit `http://localhost:3000/some-protected-page`. Sign in with Google.
**Expected:** After login, user is redirected back to `/some-protected-page` (not to `/`). The URL in the browser matches the original destination.
**Why human:** Requires browser navigation flow testing with callbackUrl parameter tracking.

---

## Gaps Summary

No gaps found. All artifacts are present, substantive, and wired. All key links connect correctly. All 5 AUTH requirements are covered by the three plans and their implementations match what was specified.

The 4 human verification items are all flow-level confirmations that require live Google credentials and browser interaction. The SUMMARY.md documents that Task 4 of Plan 02-03 was a human checkpoint that was executed and approved during the implementation session (the admin seed email was updated during that verification), but independent re-verification is listed here as required per standard protocol.

---

_Verified: 2026-03-01_
_Verifier: Claude (gsd-verifier)_
