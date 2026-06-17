# Codebase Concerns

**Analysis Date:** 2026-02-28

## Tech Debt

**Backend Scaffold Incomplete:**
- Issue: `hefesto-back/` exists only as empty repo with `.git/`, `.gitignore`, and blank `README.md`. NestJS scaffold not yet created.
- Files: `hefesto-back/` (empty)
- Impact: Blocks Phases 2-7. All backend functionality (auth, insumos, productos, calculadora) cannot be implemented until scaffold complete.
- Fix approach: Execute Phase 2 plan from `.claude/plans/hefesto-back/fase-2-scaffold-backend.md` — scaffold NestJS with TypeORM, migrations, Docker, CORS, Helmet, rate limiting.

**NextAuth Not Configured:**
- Issue: Frontend has `.env.example` with `NEXTAUTH_URL` and `NEXTAUTH_SECRET`, but no NextAuth implementation exists (`src/app/api/auth/` missing).
- Files: `hefesto-front/src/app/page.tsx`, `hefesto-front/.env.example`
- Impact: Google OAuth flow cannot work. Auth endpoints don't exist. Blocks Phase 3 entirely.
- Fix approach: Implement NextAuth configuration during Phase 3 — create `src/app/api/auth/[...nextauth].ts` with Google provider, JWT callback, session validation.

**Auth Guards Not Implemented:**
- Issue: Phase 2 scaffold plan explicitly marks with TODO comment that `AuthGuard` placeholder will be in `src/common/common.module.ts` — auth validation logic deferred to Phase 3.
- Files: `.claude/plans/hefesto-back/fase-2-scaffold-backend.md` (line 154), future `hefesto-back/src/common/common.module.ts`
- Impact: All Phase 3+ endpoints will be unprotected until auth guards are implemented.
- Fix approach: Phase 3 will implement `AuthModule` with JWT strategy, role-based decorators (`@RequireRole()`, `@Public()`), and integrate guard in `AppModule`.

**Database Entities Not Created:**
- Issue: PostgreSQL schema defined in `scope-v1.md` (13 tables) but no TypeORM entities exist yet. TypeORM migrations configured but empty.
- Files: `hefesto-back/src/entities/` (doesn't exist yet)
- Impact: Cannot run migrations. Cannot build queries for insumos, productos, etc.
- Fix approach: Create entities in Phase 4 onwards as each module is implemented (supplies in Phase 4, products in Phase 5, config in Phase 6).

## Known Bugs

**Theme Hydration Hazard (Potential):**
- Symptoms: Possible hydration mismatch between server and client theme rendering in dark mode
- Files: `hefesto-front/src/app/layout.tsx` (line 48), `hefesto-front/src/app/page.tsx` (lines 20-22)
- Trigger: Page load with `suppressHydrationWarning` set, but custom `useMounted()` hook using `useSyncExternalStore` with different server/client snapshots
- Workaround: Wrapping in `{mounted &&}` condition prevents rendering mismatch, but `suppressHydrationWarning` is a code smell
- Current state: Working as intended but uses deprecated pattern. Should migrate to next-themes v0.3+ built-in hydration handling.

**Incomplete Test Coverage:**
- Symptoms: Only one test file exists (`page.test.tsx`), mocking `next-themes` but no actual authentication, API integration, or business logic tests
- Files: `hefesto-front/src/app/__tests__/page.test.tsx`
- Trigger: Any change to theme logic or component structure without corresponding test updates will break in CI
- Workaround: Current tests are shallow and pass, masking lack of integration testing
- Fix approach: Add test fixtures in Phase 3 (auth tests), Phase 4 (API mocking), Phase 6 (calculator tests)

## Security Considerations

**Environment Variables Not Protected:**
- Risk: `.env.example` contains `NEXT_PUBLIC_API_URL=http://localhost:4000` (safe) but also `NEXTAUTH_SECRET=generate-with-openssl-rand-base64-64` (placeholder). Developers may forget to generate real secret.
- Files: `hefesto-front/.env.example`
- Current mitigation: Example value clearly marked as placeholder
- Recommendations: Add to pre-commit hook validation to block commits with `NEXTAUTH_SECRET=generate-with-openssl-rand-base64-64` in actual `.env` file. Document in Phase 1 completion checklist.

**CORS Not Yet Configured:**
- Risk: Frontend (Vercel: `vercel.app` domain) will make requests to Backend (Railway: `railway.app` domain). Without CORS headers, all API calls will fail in production.
- Files: `hefesto-back/` (future config in `src/main.ts`)
- Current mitigation: Localhost dev has same-origin for testing
- Recommendations: Phase 2 scaffold must include `.env.example` with `FRONTEND_URL` and configure `app.enableCors()` with explicit origin whitelist. Never use `credentials: true` with wildcard origins.

**Google OAuth Credentials Exposure Risk:**
- Risk: Google OAuth client ID will be in `NEXT_PUBLIC_*` env vars (must be public), but callback URL must be exactly configured in Google Cloud Console. Misconfiguration allows CSRF/token theft.
- Files: Future Phase 3 implementation in `hefesto-front/src/app/api/auth/[...nextauth].ts`
- Current mitigation: Not yet implemented
- Recommendations: Document Google Cloud setup in `.claude/docs/` before Phase 3. Require redirect URI validation. Use state parameter (NextAuth handles this). Test callback URL in staging before production deploy.

**Database Credentials in Docker:**
- Risk: `docker-compose.yml` (to be created in Phase 2) will contain PostgreSQL password in plain text
- Files: Future `hefesto-back/docker-compose.yml`
- Current mitigation: Local dev only, credentials never pushed to git
- Recommendations: `.env` file for Docker Compose (sourced via `.env` in compose file). Document in `.gitignore` maintenance. Production database accessed via Railway UI, not local.

**No Rate Limiting on Frontend:**
- Risk: Frontend submitting requests without built-in rate limiting. Backend rate limiting alone can't prevent rapid client-side requests (DOS from single user).
- Files: `hefesto-front/src/` (future API client)
- Current mitigation: Helmet + rate limiting on backend (Phase 2)
- Recommendations: Implement request debouncing in React hooks, exponential backoff on retries. Add client-side validation before API calls.

## Performance Bottlenecks

**Unnecessary Re-renders on Theme Toggle:**
- Problem: `useMounted()` hook returns different values on server vs client, causing full component re-render. `mounted` state changes trigger re-render, then `theme` state changes trigger another.
- Files: `hefesto-front/src/app/page.tsx` (lines 20-30)
- Cause: Two state changes (mounted → theme) in same effect lifecycle
- Improvement path: Use `next-themes` built-in `useTheme().resolvedTheme` instead of custom hook. Eliminates double render.

**Missing Image Optimization:**
- Problem: Frontend has fonts loaded from Google Fonts + local fonts, but no mention of image optimization for product photos (Hefesto business = leather products, needs product images)
- Files: `hefesto-front/src/app/layout.tsx` (font loading), no `<Image>` component usage yet
- Cause: Early stage (scaffold), but will be needed in Phase 5 (products) and V2 (product gallery)
- Improvement path: Plan for `next/image` integration in Phase 5 product pages. Predefine image CDN or local storage strategy.

**Frontend Bundle Size Not Monitored:**
- Problem: 30+ dev dependencies installed including `@testing-library/*`, `eslint-plugin-sonarjs`, `prettier-plugin-tailwindcss`. Bundle size not tracked in CI.
- Files: `hefesto-front/package.json` (lines 33-59)
- Cause: No `size-limit` or `bundleanalyzer` in dev toolchain
- Improvement path: Add `next/bundle-analyzer` to next.config or `@next/bundle-analyzer` for tracking. Set thresholds in CI to warn on increase > 10%.

**Database Query Performance Not Profiled:**
- Problem: TypeORM entities not yet created, migrations not written. When they are (Phase 4+), no mention of query performance testing or index strategy.
- Files: Future entities in `hefesto-back/src/entities/`
- Cause: Data model defined in `scope-v1.md` but no profiling plan
- Improvement path: Phase 4 migration creation should include index definitions (e.g., `(supply_id, date DESC)` on `supplies_price_history` as noted in `scope-v1.md` line 42). Profile queries with slow-log in local Docker.

## Fragile Areas

**`useMounted()` Hook Implementation:**
- Files: `hefesto-front/src/app/page.tsx` (lines 8-22)
- Why fragile: Custom implementation of hydration safety using `useSyncExternalStore` with hardcoded snapshots. If someone refactors theme logic or adds `useEffect` hooks, easy to introduce hydration mismatches.
- Safe modification: Don't modify. Replace entirely with `next-themes` v0.3+ built-in hydration in Phase 3. If must modify before then, ensure `getServerSnapshot()` always matches server-side behavior.
- Test coverage: No tests for hydration mismatch scenarios. Unit tests pass but don't cover mismatch.

**ESLint `@typescript-eslint/explicit-function-return-type` Exception:**
- Files: `hefesto-front/eslint.config.mjs` (lines 47-48)
- Why fragile: Shadcn/ui components in `src/components/ui/**/*.tsx` have explicit return type disabled. If code is moved outside `ui/` directory, rule suddenly applies and build breaks.
- Safe modification: Keep exception narrow (only `ui/` directory). Add comment explaining why (Shadcn convention). If refactoring UI components, update eslint rule path.
- Test coverage: No linting test to catch when files move.

**Jest Coverage Exclusions Too Broad:**
- Files: `hefesto-front/jest.config.js` (lines 16-20)
- Why fragile: Excludes entire `src/**/index.ts` files and `.d.ts`. If someone creates barrel exports for business logic in `src/lib/index.ts`, they're uncovered.
- Safe modification: Restrict exclusions to specific paths: `src/components/ui/` and `src/lib/utils.ts` only. Everything else should be tested.
- Test coverage: No validation that coverage threshold is actually being applied.

**Hard-coded Localhost URLs:**
- Files: `hefesto-front/.env.example` (line 2: `NEXT_PUBLIC_API_URL=http://localhost:4000`), `hefesto-front/src/app/layout.tsx` metadata
- Why fragile: When deploying to staging/production, `.env` files must match (Vercel ↔ Railway domains). No automation to validate they're aligned.
- Safe modification: Document in `.claude/docs/deployment.md` (to be created) the exact steps: (1) get backend URL from Railway UI, (2) create `.env.production` in root dir, (3) link to Vercel env vars, (4) test health endpoint before pushing.
- Test coverage: Integration tests should validate API_URL accessibility (not yet implemented).

## Scaling Limits

**PostgreSQL Disk on Railway:**
- Current capacity: Not yet known (Phase 2 will set up)
- Limit: Railway free tier has limited storage. Hefesto V1 has 13 tables with price history (potentially millions of rows for yearly pricing). No archive/cleanup policy.
- Scaling path: Monitor `supplies_price_history` row count. After 12 months, implement partition by date or archive to S3. Include in Phase 4 when pricing module goes live.

**Frontend Bundle with Shadcn/ui:**
- Current capacity: ~878 lines of source code + 30+ dev dependencies
- Limit: Next.js 16 with Tailwind v4 + Shadcn/ui components can create large initial bundles if all 50+ components are imported. Pages/products table (Phase 5) will need many components (tables, dialogs, forms).
- Scaling path: Implement lazy loading for feature pages (Phase 3 onwards). Use dynamic imports for modals. Monitor bundle size with `next/bundle-analyzer`.

**Rate Limiting Capacity:**
- Current capacity: Not configured yet (Phase 2 will add helmet + throttler)
- Limit: Backend rate limiting applies per-IP. If behind Railway load balancer, all traffic appears from same IP.
- Scaling path: Phase 2 must configure `ThrottlerBehindProxyGuard` (already noted in plan line 154) to read X-Forwarded-For header. Test with `NODE_ENV=test` setting `THROTTLER_SKIP_IF=test`.

## Dependencies at Risk

**Tailwind CSS v4 Transition:**
- Risk: Project uses `@tailwindcss/postcss` v4 (alpha/beta). Breaking changes possible before stable 4.0.
- Files: `hefesto-front/package.json` (line 35: `"@tailwindcss/postcss": "^4"`), `hefesto-front/tailwind.config.ts` (to be created)
- Impact: Build failures, breaking CSS if major version bump changes syntax
- Migration plan: Pin Tailwind to current stable version (3.x) until v4 is GA, then evaluate breaking changes. Alternatively, commit to v4 beta and monitor changelogs weekly during Phase 2-6 development.

**React 19 + Next 16 Bleeding Edge:**
- Risk: React 19.2.3 and Next.js 16.1.6 are latest versions. Potential for edge case bugs and performance regressions.
- Files: `hefesto-front/package.json` (lines 22, 25: `"next": "16.1.6"`, `"react": "19.2.3"`)
- Impact: Breaking changes in point releases, especially with experimental features like Suspense, useTransition
- Migration plan: Monitor GitHub issues for React/Next.js with keywords "regression", "hydration", "performance". Pin to stable versions if issues arise. Consider using LTS versions if available.

**TypeORM with PostgreSQL:**
- Risk: TypeORM is widely used but actively maintained with breaking changes. PostgreSQL compatibility depends on both.
- Files: Future `hefesto-back/package.json` (to be added in Phase 2)
- Impact: Migration scripts may break between versions, entity decorators may need updates
- Migration plan: Pin TypeORM to v0.3.x (mature release) rather than latest. Review changelogs before updating. Test migrations locally before deploying.

**Husky Git Hooks:**
- Risk: Husky v9 (installed in Phase 1) has had issues with Windows Git bash, permission errors on pre-commit hooks.
- Files: `hefesto-front/.husky/pre-commit`, `hefesto-front/package.json` (line 50: `"husky": "^9.1.7"`)
- Impact: Developers on Windows may have pre-commit hook failures preventing commits
- Migration plan: Monitor Husky v9 releases. If issues occur, add troubleshooting to `.claude/docs/setup.md`. Alternatively, use `lint-staged` wrapper around Husky to avoid full lint on every commit.

## Missing Critical Features

**No CI/CD Pipeline:**
- Problem: No `.github/workflows/` directory. No automated linting, testing, or build verification on PR.
- Blocks: Quality assurance. PRs can be merged with lint errors, failing tests, or broken builds.
- Solution: Create `.github/workflows/` in Phase 1 follow-up (after current scaffold). Add `test.yml` (lint + test + build) and `deploy.yml` (build + deploy to Vercel/Railway on main branch).

**No Swagger/API Documentation:**
- Problem: Backend (Phase 2) will have @nestjs/swagger configured, but no setup for frontend to discover endpoints.
- Blocks: Phases 3-6 depend on clear API contracts. Without Swagger, frontend devs guess endpoint signatures.
- Solution: Phase 2 scaffold includes Swagger setup (`GET /docs`). Document in Phase 2 checklist: verify Swagger runs on `localhost:4000/docs` and lists all endpoints.

**No Database Seed Script:**
- Problem: Review notes (line 37-40) explicitly flag this: without seed data, can't test until Phase 7 (data migration).
- Blocks: Phases 3-6 frontend integration. Dev database is empty, no realistic data to test against.
- Solution: Phase 4 (when first entities exist) create `scripts/seed.ts` with realistic data: 5 suppliers, 10 supplies, 5 product types, 20 products. Document in Phase 4 plan.

**No Deployment Documentation:**
- Problem: Vercel (frontend) and Railway (backend) deployment process not documented in `.claude/docs/`.
- Blocks: Phase 7 (and beyond). Users unfamiliar with Railway + Vercel will struggle to deploy.
- Solution: Create `.claude/docs/deployment.md` before Phase 2 completes. Include: (1) Railway PostgreSQL setup, (2) .env configuration, (3) Vercel frontend linking, (4) health check validation.

## Test Coverage Gaps

**No API Integration Tests:**
- What's not tested: Frontend-to-backend communication. API client functions, response handling, error scenarios.
- Files: `hefesto-front/src/` (no `api/` or `hooks/useApi.ts` yet)
- Risk: Frontend pages may call wrong endpoints, pass wrong data types, or fail silently on API errors.
- Priority: High — must implement in Phase 3 (auth) and Phase 4 (insumos integration)

**No Calculator Tests:**
- What's not tested: Cost calculation logic, pricing formulas, Tiendanube commission math.
- Files: `hefesto-back/src/` (modules not created yet, but calculadora prototipo exists at root: `calculadora/calculadora-tiendanube.jsx`)
- Risk: Wrong calculations shipped to production. Hefesto business model depends on accurate pricing.
- Priority: High — Phase 6 must include unit tests for `calcForward`, `calcInverse`, IVA, IIBB calculations. Reference test data from `Costos Hefesto.xlsx`.

**No E2E Tests:**
- What's not tested: Full user workflows (login → add supply → create product → calculate price)
- Files: No playwright/cypress configuration
- Risk: Regressions in multi-step flows. UI changes break API integration without notice.
- Priority: Medium — implement after Phase 6 if time permits. Use playwright (lighter than cypress). Test critical flows: auth login, add/edit supplies, create product.

**No Database Migration Tests:**
- What's not tested: TypeORM migrations run cleanly, data integrity preserved on version updates.
- Files: Future `hefesto-back/src/migrations/` (created in Phase 4 onwards)
- Risk: Production database corruption on deploy. Rollback not tested.
- Priority: Medium — Phase 4+ must include migration test script. Run `npm run migration:revert` and `npm run migration:run` in CI to validate idempotency.

**No Backend Unit Tests:**
- What's not tested: Service logic (cost calculations, price lookups), entity validations.
- Files: Future modules in `hefesto-back/src/{supplies,products,config}/`
- Risk: Backend logic errors not caught until integration testing.
- Priority: High — Phase 2 scaffold must include Jest setup. Phase 3+ implement tests for each service before endpoints.

---

*Concerns audit: 2026-02-28*
