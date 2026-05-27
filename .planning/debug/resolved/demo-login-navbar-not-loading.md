---
status: resolved
trigger: "al usar demologin en localhost no carga el navbar, solo al reiniciar la página aparece"
created: 2026-05-27T00:00:00Z
updated: 2026-05-27T00:00:00Z
resolved: 2026-05-27T00:00:00Z
---

## Resolution

root_cause: The demo (credentials) login used a Server Action `signIn('credentials', { redirectTo: '/' })` (server-side `signIn` from `@/auth`) in `DemoLoginButton.tsx`. This performs an INTERNAL soft navigation `/login` → `/`. The `SessionProvider` is mounted once in the ROOT layout with no initial `session` prop, so the client `useSession()` keeps the unauthenticated session it cached while on `/login` and never re-fetches on the soft nav. The sidebar (`AppSidebar` → `usePermissions()` → `useSession()` → `NO_PERMISSIONS`) therefore hides every permission-gated nav group until a hard reload. Google login was unaffected because its OAuth round-trip is a hard external navigation that remounts/refetches the provider. The code comment claiming the (auth) layout "has no SessionProvider" was false — SessionProvider is mounted at the root layout, which wraps /login.

fix: Converted `nemea-front/src/app/(auth)/login/DemoLoginButton.tsx` to a client component (`'use client'`) that calls the client-side `signIn` from `next-auth/react` with `{ redirect: false }`, then navigates with `router.push('/')` + `router.refresh()`. The client `signIn` updates the SessionProvider session directly, so the sidebar populates after one click without a hard reload. Removed the misleading "(auth) layout has no SessionProvider" comment. Scope limited to Option A — `app/layout.tsx` and `providers/SessionProvider.tsx` were NOT touched.

verification:
- `npx tsc --noEmit`: PASS (no errors)
- `npm run lint`: PASS (0 errors; 4 pre-existing `react-hooks/incompatible-library` warnings in unrelated files — usuarios/expenses/products — unchanged by this fix)
- `npm test`: PASS (5 suites, 24 tests; pre-existing `act(...)` console noise in CalculadoraClient test, no failures)
- HUMAN VERIFY (requires running app): with `NEXT_PUBLIC_DEMO_LOGIN_ENABLED=true`, on /login click "Entrar como demo" → confirm the sidebar shows all permission-gated groups (Finanzas, Herramientas, Productos, Datos base, Admin) immediately, with no hard reload.

specialist_review: typescript-expert (next-auth v5 client signIn pattern) — see Specialist Review section.

commit: see fix(auth) commit in nemea-front sub-repo.

## Specialist Review

specialist: typescript-expert (next-auth v5 client signIn pattern)
result: LOOKS_GOOD
notes: The fix follows the canonical next-auth v5 (beta) credentials pattern for SPA-style login without a full reload — client signIn('credentials', { redirect: false }) resolves the session into the SessionProvider, then router.push('/') + router.refresh() performs a soft client navigation that re-runs server components with the now-authenticated session. useSession() (and thus usePermissions() and the sidebar) reads the updated client session immediately. Error handling guards result?.error before navigating, and an isPending guard prevents double-submit. No idiomatic pitfalls flagged: the awaited async handler is correctly fire-and-forgotten via void in the onClick, button is type='button' (no stray form submit), and no any casts were introduced. Scope is minimal — only DemoLoginButton.tsx changed.

## Current Focus

hypothesis: CONFIRMED. The demo (credentials) login calls `signIn('credentials', { redirectTo: '/' })` from a Server Action (`DemoLoginButton.tsx`), which performs an INTERNAL (soft) navigation `/login` → `/`. The `SessionProvider` is mounted once in the ROOT layout (`app/layout.tsx`), shared across `/login` and `/`, and is NOT given an initial `session` prop — so the client `useSession()` caches the unauthenticated session it fetched while on `/login` and never re-fetches on the soft nav. The sidebar (`AppSidebar`, a client component) reads permissions via `usePermissions()` → `useSession()` → `NO_PERMISSIONS`, so every permission-gated nav group is hidden (only ungated "Inicio" shows). The Home page cards DO render because `Home` is a Server Component reading `auth()` directly. Google login works because `signIn('google')` forces a HARD external browser round-trip (consent screen + `/api/auth/callback/google`) that remounts the SessionProvider and re-fetches the session. A hard reload / browser restart "fixes" the demo case for the same reason.
test: Confirm the mechanism with the app running — after demo login, check whether `/api/auth/session` returns populated permissions while `useSession()` in the client still shows empty until a hard reload. Then validate the fix (client-side `signIn` from `next-auth/react` on the demo button, or seed `SessionProvider` with the server session + force refresh) makes the sidebar populate without reload.
expecting: With the fix, after one click on "Entrar como demo" the sidebar shows all permission-gated groups immediately, no reload.
next_action: RESOLVED via Option A (client signIn). Awaiting human functional verification on the running app.

## Symptoms

expected: After clicking "Entrar como demo" on /login (demo flags enabled), land on / authenticated as the demo ADMIN user with the sidebar fully populated (Finanzas, Herramientas, Productos, Datos base, Admin groups all visible).
actual: Sidebar shows only "Inicio" (the one ungated item). The permission-gated nav groups do not appear until a hard reload / browser restart. The Home page permission cards (Productos, Calculadora, Insumos, Gastos) DO render correctly on first load.
errors: No console errors reported by the user. To be confirmed during investigation.
reproduction: localhost. Set DEMO_LOGIN_ENABLED=true (nemea-back/.env) and NEXT_PUBLIC_DEMO_LOGIN_ENABLED=true (nemea-front/.env.local), run migrations (demo user seeded), bring up back + front, go to /login, click "Entrar como demo". Sidebar nav groups missing until hard reload.
started: Surfaces now with the Phase 12.5 demo (credentials) login. Google login does NOT exhibit the bug — confirmed by the user. The underlying SessionProvider/layout setup is pre-existing (not touched by 12.5); the credentials soft-nav login is what triggers it.

## Eliminated

- hypothesis: Server-side session/permissions are wrong for the demo user
  evidence: The Home page Server Component (reads `auth()` server-side) renders all permission cards correctly on first load, so the backend JWT and server session DO carry the demo user's ADMIN permissions. The problem is client-side session propagation, not the server session or the demo user's role.
  timestamp: 2026-05-27T00:00:00Z

- hypothesis: Pure SessionProvider-missing-initial-session gap (would affect all logins)
  evidence: Google login does NOT show the bug. A bug in the shared SessionProvider seeding alone would affect Google identically. The differentiator is the navigation type: Google = hard external OAuth round-trip (remounts/refetches); demo credentials = soft internal redirect (no remount/refetch).
  timestamp: 2026-05-27T00:00:00Z

## Evidence

- timestamp: 2026-05-27T00:00:00Z
  checked: nemea-front/src/components/layout/AppSidebar.tsx
  found: Client component ('use client'). Nav groups are gated by `usePermissions()` destructured flags (canViewExpenses, canUseCalculator, canManageScenarios, canViewProducts, canViewSupplies, canManageUsers, canManageConfig). Only TOP_ITEMS ("Inicio") is ungated.
  implication: If permissions resolve to NO_PERMISSIONS client-side, only "Inicio" renders — exactly the observed symptom.

- timestamp: 2026-05-27T00:00:00Z
  checked: nemea-front/src/hooks/usePermissions.ts
  found: `const { data: session } = useSession(); return session?.user?.permissions ?? NO_PERMISSIONS;`
  implication: Sidebar permissions depend entirely on the client `useSession()` value. If the client session is empty/stale, all gated groups disappear.

- timestamp: 2026-05-27T00:00:00Z
  checked: nemea-front/src/app/layout.tsx + nemea-front/src/providers/SessionProvider.tsx
  found: Root layout is a synchronous (non-async) function mounting `<SessionProvider>` with NO `session` prop. The wrapper forwards only `children` to NextAuth's `SessionProvider`. So the client must fetch `/api/auth/session` itself; there is no server-seeded initial session.
  implication: useSession starts empty and relies on client fetch + its cache. Combined with a soft nav that doesn't remount the provider, the empty cached session persists.

- timestamp: 2026-05-27T00:00:00Z
  checked: nemea-front/src/app/(auth)/login/DemoLoginButton.tsx and login/page.tsx
  found: BOTH logins use server-side `signIn` from `@/auth`. Demo: `signIn('credentials', { email: 'demo@nemea.app', redirectTo: '/' })` (internal redirect). Google: `signIn('google', { redirectTo })` (external OAuth redirect). DemoLoginButton has a code comment claiming it uses a Server Action "because the (auth) layout has no SessionProvider" — but SessionProvider is mounted at the ROOT layout, which wraps /login too, so that premise appears incorrect.
  implication: The Google vs demo difference is the navigation type (hard external vs soft internal), supporting the hypothesis. It also means a client-side `signIn` from next-auth/react on the demo button is likely available (SessionProvider is in scope at /login) and would update useSession directly.

- timestamp: 2026-05-27T00:00:00Z
  checked: nemea-front/src/auth.ts (jwt/session callbacks)
  found: credentials branch of `jwt` sets token.backendToken/permissions/userId from the `user` arg; session callback maps token.permissions → session.user.permissions ?? NO_PERMISSIONS. The stale-JWT block only fires when token has legacy `role` and no `permissions` (not the fresh-demo case). Code review WR-01/WR-02 flagged unsafe casts + stale-token handling here, but these are not obviously the cause of the navbar symptom.
  implication: The token/session shape is correct on the server; the gap is client-side propagation after the soft-nav credentials login.

- timestamp: 2026-05-27T00:00:00Z
  checked: Applied fix — converted DemoLoginButton.tsx to client component using next-auth/react signIn ({ redirect: false }) + router.push('/') + router.refresh().
  found: tsc --noEmit clean; lint 0 errors; jest 24/24 pass. No regressions in the existing suite.
  implication: Static + test verification clears. Functional confirmation (sidebar populates without reload) handed to user as a human-verify step.
