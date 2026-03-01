---
phase: 02-auth
plan: 03
subsystem: auth
tags: [nextauth-v5, google-oauth, proxy-ts, session-provider, jwt-session, login-page, access-denied, header-dropdown, useIsAdmin]

# Dependency graph
requires:
  - phase: 02-auth plan 01
    provides: User entity, UsersService, JwtAuthGuard, RolesGuard, AuthModule, decorators
  - phase: 02-auth plan 02
    provides: AuthService with validateGoogleToken, AuthController (POST /auth/google, GET /auth/me), UsersController
provides:
  - NextAuth v5 config with Google provider and JWT/session callbacks exchanging tokens with NestJS backend
  - proxy.ts route protection redirecting unauthenticated users to /login with callbackUrl
  - proxy.ts redirect for authenticated users without backendToken to /acceso-denegado
  - SessionProvider wrapping app for client-side session access
  - apiFetch wrapper attaching Authorization Bearer header from session
  - TypeScript module augmentation for Session (accessToken, role, id) and JWT (backendToken, role, userId)
  - Branded /login page with Nemea engraving font and Google sign-in button
  - /acceso-denegado page with friendly message and logout button
  - Header component with sticky bar, Nemea branding, and user avatar dropdown (name, email, logout)
  - useIsAdmin() hook for Phase 3+ role-gating (hidden not disabled pattern)
  - Shadcn/ui DropdownMenu and Avatar components installed
affects: [03-catalogs, 04-supplies, 05-products, 06-costs, 07-expenses]

# Tech tracking
tech-stack:
  added: [next-auth@5.0.0-beta.30]
  patterns: [nextauth-v5-google-provider, jwt-callback-token-exchange, proxy-ts-route-protection, server-action-sign-in, session-provider-wrapper, api-fetch-bearer-wrapper, use-is-admin-hook]

key-files:
  created: ["nemea-front/src/auth.ts", "nemea-front/src/proxy.ts", "nemea-front/src/app/api/auth/[...nextauth]/route.ts", "nemea-front/src/providers/SessionProvider.tsx", "nemea-front/src/lib/api.ts", "nemea-front/src/types/next-auth.d.ts", "nemea-front/src/app/login/page.tsx", "nemea-front/src/app/acceso-denegado/page.tsx", "nemea-front/src/components/layout/Header.tsx", "nemea-front/src/hooks/useIsAdmin.ts", "nemea-front/src/components/ui/dropdown-menu.tsx", "nemea-front/src/components/ui/avatar.tsx"]
  modified: ["nemea-front/src/app/layout.tsx", "nemea-front/package.json", "nemea-front/.env.example", "nemea-front/.gitignore", "nemea-back/src/database/migrations/1772369233961-CreateUserTable.ts"]

key-decisions:
  - "Backend call only in jwt callback (not signIn) -- avoids double POST /auth/google per Pitfall 2"
  - "AUTH_SECRET env var (not NEXTAUTH_SECRET) -- Auth.js v5 auto-detects AUTH_SECRET per Pitfall 5"
  - "proxy.ts matcher excludes api routes -- prevents NextAuth route handler interception per Pitfall 6"
  - "Login page uses server action (signIn from @/auth) -- avoids client component for the entire page"
  - "Access-denied page is client component -- signOut from next-auth/react requires use client"
  - "Header returns null when no session -- ensures header only renders on authenticated pages"
  - ".env.example tracked via !.env.example gitignore exception -- .env* pattern was blocking it"
  - "Updated .env.example replacing deprecated NEXTAUTH_SECRET/NEXTAUTH_URL with AUTH_SECRET/AUTH_GOOGLE_ID/AUTH_GOOGLE_SECRET"

patterns-established:
  - "NextAuth v5 token exchange: jwt callback calls POST /api/auth/google with account.id_token, stores backendToken/role/userId in JWT"
  - "proxy.ts route protection: unauthenticated -> /login with callbackUrl; authenticated without backendToken -> /acceso-denegado"
  - "apiFetch wrapper: server-side utility calling auth() to get session, attaching Authorization Bearer header"
  - "useIsAdmin hook: client-side role check for conditional UI rendering (hidden, not disabled)"
  - "Header avatar dropdown: useSession for user data, signOut for logout, renders nothing without session"

requirements-completed: [AUTH-01, AUTH-02, AUTH-04]

# Metrics
duration: 10min
completed: 2026-03-01
---

# Phase 2 Plan 03: Frontend Auth Integration Summary

**NextAuth v5 with Google provider exchanging tokens with NestJS backend, branded login/access-denied pages, proxy.ts route protection with callbackUrl, header with user avatar dropdown for logout, and E2E verified auth flow**

## Performance

- **Duration:** 10 min (7 min coding + 3 min verification/fix)
- **Started:** 2026-03-01T13:08:54Z
- **Completed:** 2026-03-01T14:05:26Z
- **Tasks:** 4 of 4
- **Files modified:** 17 (12 created, 5 modified)

## Accomplishments
- Auth.js v5 config with Google provider, JWT strategy, and callbacks that exchange Google id_token with NestJS backend POST /api/auth/google
- proxy.ts route protection: unauthenticated users redirected to /login with callbackUrl, users without backendToken redirected to /acceso-denegado
- Branded /login page with Nemea engraving font, Google sign-in button via server action, and callbackUrl redirect
- /acceso-denegado page with friendly message and logout button for non-whitelisted users
- Header component with sticky top bar, Nemea branding, and user avatar dropdown showing name/email/logout
- useIsAdmin() hook ready for Phase 3+ to conditionally hide action buttons
- apiFetch wrapper for server-side authenticated API calls with automatic Bearer token
- TypeScript module augmentation for Session and JWT types
- E2E auth flow verified by human: login, session persistence, access-denied for non-whitelisted, logout via header dropdown
- Admin seed email updated from placeholder to real user (basualdofelipe@gmail.com) during verification

## Task Commits

Each task was committed atomically:

1. **Task 1: NextAuth config, route handler, proxy, SessionProvider, types, and api wrapper** - `d3a7219` (feat, nemea-front)
2. **Task 2: Login page and access-denied page** - `f6c366a` (feat, nemea-front)
3. **Task 3: Header with user avatar dropdown and useIsAdmin hook** - `d9d5eca` (feat, nemea-front)
4. **Task 4: Verify complete auth flow end-to-end** - `ac64d1a` (fix, nemea-back) -- admin seed email updated during verification

Note: Tasks 1-3 commits are in nemea-front/ on `development`. Task 4 fix commit is in nemea-back/ on `development`.

## Files Created/Modified
- `nemea-front/src/auth.ts` - NextAuth v5 config: Google provider, JWT/session callbacks, backend token exchange
- `nemea-front/src/proxy.ts` - Route protection: unauthenticated -> /login, no backendToken -> /acceso-denegado
- `nemea-front/src/app/api/auth/[...nextauth]/route.ts` - NextAuth route handler exporting GET and POST
- `nemea-front/src/providers/SessionProvider.tsx` - Client wrapper for NextAuth SessionProvider
- `nemea-front/src/lib/api.ts` - Server-side apiFetch with automatic Bearer token from session
- `nemea-front/src/types/next-auth.d.ts` - TypeScript module augmentation for Session and JWT
- `nemea-front/src/app/login/page.tsx` - Branded login page with Nemea engraving font and Google sign-in
- `nemea-front/src/app/acceso-denegado/page.tsx` - Access-denied page with friendly message and logout
- `nemea-front/src/components/layout/Header.tsx` - Sticky header with Nemea branding and avatar dropdown
- `nemea-front/src/hooks/useIsAdmin.ts` - Client hook returning boolean for ADMIN role check
- `nemea-front/src/components/ui/dropdown-menu.tsx` - Shadcn/ui DropdownMenu component
- `nemea-front/src/components/ui/avatar.tsx` - Shadcn/ui Avatar component
- `nemea-front/src/app/layout.tsx` - Added SessionProvider wrapper and Header component
- `nemea-front/package.json` - Added next-auth@5.0.0-beta.30 dependency
- `nemea-front/.env.example` - Updated with AUTH_SECRET, AUTH_GOOGLE_ID, AUTH_GOOGLE_SECRET
- `nemea-front/.gitignore` - Added !.env.example exception to track env template
- `nemea-back/src/database/migrations/1772369233961-CreateUserTable.ts` - Updated admin seed email from placeholder to real user

## Decisions Made
- Backend call only in jwt callback (not signIn) to avoid double POST /auth/google -- per RESEARCH.md Pitfall 2
- AUTH_SECRET env var (not NEXTAUTH_SECRET) as Auth.js v5 auto-detects it -- per RESEARCH.md Pitfall 5
- proxy.ts matcher excludes `api` routes to prevent NextAuth route handler interception -- per RESEARCH.md Pitfall 6
- Login page uses server action (signIn from @/auth) to keep the page as a server component
- Access-denied page is a client component because signOut from next-auth/react requires 'use client'
- Header returns null when no session -- ensures header only renders on authenticated pages, not on /login or /acceso-denegado
- Updated .env.example with Auth.js v5 env vars, removing deprecated NEXTAUTH_SECRET and NEXTAUTH_URL

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed .env.example not tracked due to .gitignore pattern**
- **Found during:** Task 1 (creating .env.local.example)
- **Issue:** Plan specified creating .env.local.example, but .gitignore had `.env*` pattern blocking ALL env files from tracking. Also .env.example was already present but untracked.
- **Fix:** Added `!.env.example` exception to .gitignore. Updated existing .env.example with auth vars instead of creating separate .env.local.example. Removed the gitignored .env.local.example.
- **Files modified:** nemea-front/.gitignore, nemea-front/.env.example
- **Verification:** git status shows .env.example as tracked; git check-ignore confirms it's no longer ignored
- **Committed in:** d3a7219 (Task 1 commit)

**2. [Rule 1 - Bug] Fixed Shadcn/ui generated files using double quotes**
- **Found during:** Task 3 (installing DropdownMenu and Avatar)
- **Issue:** Shadcn/ui generates components with double quotes, but project ESLint config enforces single quotes
- **Fix:** Ran prettier --write on avatar.tsx and dropdown-menu.tsx to convert to single quotes
- **Files modified:** nemea-front/src/components/ui/avatar.tsx, nemea-front/src/components/ui/dropdown-menu.tsx
- **Verification:** npm run lint passes with 0 errors
- **Committed in:** d9d5eca (Task 3 commit)

---

**Total deviations:** 2 auto-fixed (1 blocking, 1 bug)
**Impact on plan:** Both fixes necessary for correctness. No scope creep.

## Issues Encountered
- Pre-existing npm audit shows 2 high severity vulnerabilities (not introduced by this plan, pre-existing in Next.js ecosystem)

## User Setup Required

Completed during Task 4 verification:
1. `.env.local` created in `nemea-front/` with AUTH_SECRET, AUTH_GOOGLE_ID, AUTH_GOOGLE_SECRET, NEXT_PUBLIC_API_URL
2. `http://localhost:3000/api/auth/callback/google` added to Google OAuth Client's Authorized redirect URIs
3. Backend GOOGLE_CLIENT_ID updated to match frontend AUTH_GOOGLE_ID
4. Admin seed email updated to `basualdofelipe@gmail.com` (real user)

## Next Phase Readiness
- Phase 2 Auth is fully complete -- all auth flows verified end-to-end by human tester
- Admin seed uses real email (basualdofelipe@gmail.com), verified working with Google OAuth
- apiFetch wrapper ready for all future data-fetching in Phase 3+
- useIsAdmin() hook ready for Phase 3+ role-gating
- Header with logout ready for all authenticated pages
- Ready to proceed to Phase 3: Catalogs and Suppliers

## Self-Check: PASSED

All 12 created files and 5 modified files verified on disk. All 4 commit hashes (d3a7219, f6c366a, d9d5eca, ac64d1a) verified in git log. Build passes (0 errors). Lint passes (0 errors). E2E auth flow verified by human. SUMMARY.md exists.

---
*Phase: 02-auth*
*Completed: 2026-03-01*
