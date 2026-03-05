---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: unknown
last_updated: "2026-03-05T22:36:39.879Z"
progress:
  total_phases: 4
  completed_phases: 4
  total_plans: 13
  completed_plans: 13
---

---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: phase_complete
last_updated: "2026-03-05T22:55:00Z"
progress:
  total_phases: 4
  completed_phases: 4
  total_plans: 12
  completed_plans: 12
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-28)

**Core value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automáticamente cuando cambian los precios de los insumos.
**Current focus:** Phase 4 complete — Supplies and Price History fully verified. Next: Phase 5 Products and BOM

## Current Position

Phase: 4 of 7 (Supplies and Price History) -- COMPLETE
Plan: 3 of 3 in current phase (all complete, all gaps closed)
Status: Phase 04 fully complete. All 3 plans executed, verification passed 7/7, gap closure plan 04-03 fixed all 3 gaps.
Last activity: 2026-03-05 — Plan 04-03 gap closure complete (supplier validation + form reset + Fragment key)

Progress: [############░] 52%

## Performance Metrics

**Velocity:**
- Total plans completed: 12
- Average duration: 7.8min
- Total execution time: 1.56 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1 - Backend Scaffold | 3 | 20min | 6.7min |
| 2 - Auth | 3 | 24min | 8min |
| 3 - Catalogs & Suppliers | 4/4 | 40min | 10min |
| 4 - Supplies & Price History | 3/3 | 27min | 9min |

**Recent Trend:**
- Last 5 plans: 03-03 (20min), 03-04 (5min), 04-01 (4min), 04-02 (8min), 04-03 (15min)
- Trend: gap closure plans take longer due to investigation + iteration

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: synchronize: false hardcoded unconditionally — never conditioned on NODE_ENV (Pitfall 1)
- Roadmap: Two TypeORM DataSource configs share the same entities array from src/database/dataSource.ts (Pitfall 2)
- Roadmap: CostsModule is separate from ProductsModule to avoid circular deps (imports both SuppliesModule + ProductsModule)
- Roadmap: TiendanubeConfigModule named explicitly to avoid collision with @nestjs/config's ConfigModule
- Roadmap: Soft delete strategy (is_active vs @DeleteDateColumn) — decision deferred to Phase 3/4 planning
- 01-01: SWC builder with typeCheck: true for fast compilation + type safety
- 01-01: ESLint flat config (not legacy .eslintrc) matching frontend sonarjs/explicit-return-type conventions
- 01-01: Port 4000 in main.ts per CLAUDE.md backend convention
- 01-01: ES2021 target in tsconfig for wider runtime compatibility (scaffold defaulted to ES2023)
- 01-02: Shared DataSource pattern -- data-source.ts is the single source of truth for CLI and NestJS runtime
- 01-02: synchronize: false unconditional, migrationsRun: true for auto-execution on app start
- 01-02: Definite assignment assertion (!) on entity properties required for TypeScript strict + TypeORM decorators
- 01-02: E2E test isolation via setup-e2e.ts overriding DATABASE_URL to test database on port 5433
- 01-03: ResponseInterceptor excludes Swagger routes from envelope wrapping to preserve raw Swagger JSON
- 01-03: Health logic kept in AppController (no delegation to AppService) since it is trivial
- 01-03: HttpExceptionFilter catches all exceptions (@Catch() with no args), not just HttpException
- 01-03: E2E tests mirror main.ts setup (prefix, pipes, filters, interceptors, helmet) for realistic testing
- 02-01: Guard registration order: ThrottlerGuard in AppModule, JwtAuthGuard first then RolesGuard in AuthModule
- 02-01: JwtStrategy checks whitelist on every request via usersService.findActiveByEmail -- instant revocation
- 02-01: JWT TTL 7 days -- whitelist check per request makes short TTL unnecessary
- 02-01: @Public() opt-out pattern for health and swagger routes
- 02-01: Admin seed in migration (updated to basualdofelipe@gmail.com during 02-03 verification)
- 02-01: google-auth-library installed for id_token verification in Plan 02-02
- 02-02: AuthService.verifyGoogleIdToken as private method -- clean mocking without overriding google-auth-library internals
- 02-02: POST /auth/google returns 200 (not 201) -- token exchange is not resource creation
- 02-02: Unknown routes return 404 (not 401) -- NestJS resolves routes before guards run on unmatched paths
- 02-02: JwtUser import type required for isolatedModules + emitDecoratorMetadata compatibility
- 02-03: Backend call only in jwt callback (not signIn) -- avoids double POST /auth/google per Pitfall 2
- 02-03: AUTH_SECRET env var (not NEXTAUTH_SECRET) -- Auth.js v5 auto-detects per Pitfall 5
- 02-03: proxy.ts matcher excludes api routes -- prevents NextAuth route handler interception per Pitfall 6
- 02-03: Login page uses server action (signIn from @/auth) -- keeps page as server component
- 02-03: Header returns null when no session -- renders only on authenticated pages
- 02-03: .env.example tracked via !.env.example gitignore exception
- 02-03: Updated .env.example replacing deprecated NEXTAUTH_SECRET with AUTH_SECRET/AUTH_GOOGLE_ID/AUTH_GOOGLE_SECRET
- 03-01: UUID migration uses add-column/drop-column strategy (not ALTER TYPE) for clean SERIAL-to-UUID conversion
- 03-01: PK constraint renamed from auto-generated hash to semantic PK_users_id
- 03-01: Consistent mock UUID (a1b2c3d4-e5f6-7890-abcd-ef1234567890) across all test files
- 03-02: Generic dimension-to-repository map in CatalogsService avoids 6 duplicate controllers for identical name-only entities
- 03-02: Partial unique index on suppliers (WHERE is_active = true) allows reactivating deactivated supplier names
- 03-02: FK-safe delete prepared with 23503 catch for future Phase 5 product references
- 03-02: Supplier toggle-status via PATCH endpoint instead of DELETE for soft delete
- 03-03: Route groups (app)/(auth) separate sidebar layout from auth pages cleanly
- 03-03: Client-side apiClientFetch wrapper mirrors server-side apiFetch for mutations
- 03-03: Inline CRUD pattern for catalogs avoids page navigation for simple name edits
- 03-03: Server-component data fetching with client-component interactivity for catalog tabs
- 03-03: EditSupplierClient wrapper bridges server-side fetch and client-side form
- 04-01: Composite partial unique index on supplies defined in migration SQL (not @Index decorator) to avoid TypeORM FK column resolution issues
- 04-01: 2-query pattern (find + DISTINCT ON) for current price instead of complex QueryBuilder subquery join
- 04-01: Supplier cascade deactivation via direct Supply repo injection in SuppliersModule (no circular dependency)
- 04-01: SupplyPriceHistory.price typed as string (TypeORM decimal behavior), frontend parses with parseFloat
- 04-01: Supplier NOT editable after supply creation (supplierId excluded from UpdateSupplyDto)
- 04-02: Sidebar restructured with SidebarGroupLabel for Datos base section (Catalogos, Proveedores, Insumos)
- 04-02: Unified zod schema for create/edit form to avoid TypeScript resolver type incompatibility with react-hook-form
- 04-02: PriceHistoryDialog uses useCallback+useRef pattern to satisfy react-hooks/set-state-in-effect lint rule
- 04-02: Shared types.ts centralizes Supply, PriceRecord, UNIT_LABELS, formatPrice for cross-component reuse
- 04-02: SupplyTypeGroup manages own collapsible state independently (all open by default)
- 04-03: ConflictException guard on toggleStatus checks both !supply.isActive AND !supply.supplier.isActive before blocking reactivation
- 04-03: useEffect deps include [supply, open, reset] to reset form on dialog reopen (not just on supply prop change)
- 04-03: Fragment with explicit key replaces React shorthand in map rendering to fix console warning

### Pending Todos

Phase 04 complete. Next: plan Phase 5 (Products and BOM).

### Known UI Issues (from 03-03 verification → 03-04 gap closure)

- ~~Supplier email validation~~ FIXED in 03-04 (empty string → undefined transform)
- Catalog tabs wrap on mobile (flex-wrap) — functional, will redesign in UI polish
- ~~Mobile sidebar backdrop~~ CONFIRMED working in 03-04

### Blockers/Concerns

- Phase 4 (Supplies): Soft delete strategy decided in 03-02 -- is_active with partial unique index. Apply same pattern to supplies.
- Phase 6 (Costs): DISTINCT ON batched query must be verified against NestJS logger to confirm exactly 2 SQL queries for the product list (explicit acceptance criterion).

## Session Continuity

Last session: 2026-03-05
Stopped at: Phase 04 complete (all 3 plans + verification + gap closure). Ready for Phase 5 planning.
Resume file: .planning/phases/04-supplies-and-price-history/04-03-SUMMARY.md
