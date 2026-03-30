---
gsd_state_version: 1.0
milestone: v1.1
milestone_name: Tiendanube & Investor Dashboard
status: executing
stopped_at: Phase 12.1 Wave 2 complete (Plans 01-03 done)
last_updated: "2026-03-30T16:30:00.000Z"
last_activity: 2026-03-30
progress:
  total_phases: 16
  completed_phases: 12
  total_plans: 37
  completed_plans: 36
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-26)

**Core value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automaticamente cuando cambian los precios de los insumos.
**Current focus:** Phase 12.1 — dynamic-roles-with-configurable-permissions

## Current Position

<<<<<<< HEAD
Phase: 12.1 (dynamic-roles-with-configurable-permissions) — EXECUTING
Plan: 3 of 4
Status: Executing Phase 12.1 (Plan 03 complete)
Last activity: 2026-03-30 -- Plan 03 frontend permission system complete
=======
Phase: 6 of 7 (Cost Calculation) -- COMPLETE
Plan: 2 of 2 in current phase (all complete)
Status: Phase complete — ready for verification
Last activity: 2026-03-30
>>>>>>> worktree-agent-a70315cb

Progress (v1.1): [####################....] 83% (5/6 phases)
Progress (overall): [#########################.] 96% (12/13 phases)

## Performance Metrics

<<<<<<< HEAD
**Velocity (from v1.0):**
=======
**Velocity:**

- Total plans completed: 18
- Average duration: 7.6min
- Total execution time: 2.16 hours
>>>>>>> worktree-agent-a70315cb

- Total plans completed: 21
- Average duration: 6.4min
- Total execution time: ~2.2 hours

**By Phase (v1.0):**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1 - Backend Scaffold | 3 | 20min | 6.7min |
| 2 - Auth | 3 | 24min | 8min |
| 3 - Catalogs & Suppliers | 4 | 40min | 10min |
| 4 - Supplies & Price History | 3 | 27min | 9min |
| 5 - Products & BOM | 4 | 31min | 7.8min |
| 6 - Cost Calculation | 2 | 11min | 5.5min |
| 7 - Expenses | 2 | ~8min | ~4min |

<<<<<<< HEAD
**v1.1 Metrics:**

| Phase | Plan | Duration | Tasks | Files |
|-------|------|----------|-------|-------|
| 8 - Hardening | 08-01 | 5min | 3 | 8 |
| 8 - Hardening | 08-02 | 9min | 2 | 22 |
| 9 - Product UX | 09-01 | 6min | 2 | 4 |
| 10 - TN Config | 10-01 | 4min | 2 | 14 |
| 10 - TN Config | 10-02 | 7min | 2 | 10 |
| 11 - Calculadora | 11-01 | 9min | 2 | 9 |
| 11 - Calculadora | 11-02 | 5min | 1+checkpoint | 8 |

*Updated after each plan completion*
| Phase 12 P01 | 8min | 3 tasks | 12 files |
| Phase 12 P02 | 5min | 3 tasks | 12 files |
| Phase 12 P03 | 3min | 2 tasks | 5 files |
| Phase 12.1 P02 | 10min | 2 tasks | 12 files |
| Phase 12.1 P03 | 9min | 2 tasks | 29 files |
=======
**Recent Trend:**

- Last 5 plans: 05-01 (5min), 05-02 (8min), 05-03 (6min), 05-04 (12min), 06-01 (5min)
- Trend: standard CRUD plans execute fast (~5min), frontend pages ~6min, wiring plans with bug fixes ~12min

*Updated after each plan completion*
| Phase 12.1 P01 | 9.5min | 2 tasks | 28 files |
>>>>>>> worktree-agent-a70315cb

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: TiendanubeConfigModule named explicitly to avoid collision with @nestjs/config's ConfigModule
<<<<<<< HEAD
- Roadmap: CostsModule is separate from ProductsModule to avoid circular deps
- v1.1: Hardening must complete before investors access the system (role enforcement gap)
- v1.1: CalculadoraService is pure backend — forward formula never duplicated in frontend
- v1.1: Scenarios use separate scenario_overrides table, never write to product_price_history
- v1.1: Zero new npm dependencies — existing stack covers all v1.1 features
- v1.1: Round financial calculations once at chain end, not at intermediate steps
- Phase 8: proxy.ts renamed to middleware.ts (export name change to `middleware`) — proxy.ts was NOT firing as Next.js middleware
- Phase 8: @Roles(Role.ADMIN) already on all mutation endpoints — HARD-03 pre-satisfied, verify only
- Phase 8: UsersController hard DELETE replaced with PATCH toggle-status (soft delete pattern)
- Phase 8: deactivateBySupplier dead code deleted (never called)
- Phase 8: SupplyOption shared type includes optional `supplier?: { name: string }` field for BOM editors
- Phase 8 (revision): 08-02 depends on 08-01 (users page calls toggle-status endpoint created in 08-01)
- Phase 8 (revision): formatDate MUST include { day: '2-digit', month: '2-digit', year: 'numeric' } options
- Phase 8 (revision): cleanSupplierData uses SupplierFormData type (not Record<string, string>)
- Phase 8 (revision): Do NOT create shared formatAmount — collision with expenses/types.ts version
- Phase 8 (revision): SupplyCombobox does not use Check/cn imports — omit from extraction
- Phase 9: BOM group editor moves from type-level to name-level (PRUX-02)
- Phase 9: Batch price button at BOTH type-level AND name-level (type = broad "all Cinturones", name = targeted "all Hercules")
- Phase 9: BOM override detection uses cost-heuristic first, then accurate detection after dialog opens
- Phase 9 (pre-mortem fix): SKU column in ProductFinishGroup is a Link to /productos/[id] — relocated from removed Nombre column
- Phase 9 (pre-mortem fix): onDivergenceDetected callback uses local `divergent` variable, not state (React batching safety)
- Phase 9 (pre-mortem fix): ProductExpandedRow colSpan explicitly 6 (matching 6-column leaf table)
- Phase 10: 5 DB tables for TN config (tn_payment_gateways, tn_gateway_rates, tn_installment_rates, tn_tax_config, tn_plans)
- Phase 10: 3 gateways modeled from day 1 (Pago Nube, Mercado Pago, MODO) — MODO seeded as inactive
- Phase 10: Append-only rate history for gateway rates, installment rates, and tax config (same pattern as supply prices)
- Phase 10: CPT depends on plan + gateway — Pago Nube always 0%, others vary by plan
- Phase 10: All rates are "% + IVA" — stored rate is base, IVA calculated on top by calculadora
- Phase 10: IIBB/SIRTAC = single configurable value (3.5%), admin enters manually
- Phase 10: Frontend config page uses Shadcn Accordion (type="multiple", all expanded by default) for collapsible sections
- Phase 10: Plan selector dropdown defaults to 'esencial' — state lifted to TiendanubeConfigClient
- Phase 10: GatewaySection uses compound component pattern (Header as static property for accordion trigger)
- Phase 10: /configuracion added to ADMIN_ONLY_ROUTES for middleware-level protection
- Phase 10 (gap closure): Raw SQL queries MUST use explicit column aliases with camelCase ("ratePercent", "paymentMethod", "withdrawalDays", "isActive") — never use SELECT * in raw SQL that feeds parse helpers
- Phase 10 (gap closure): json_build_object replaces row_to_json for gateway nested object — controls key names explicitly
- Phase 11 (planning): Multi-gateway support — all 3 pasarelas with cascading selectors
- Phase 11 (planning): Plan TN inherits from config + override toggle ("Simular otro plan")
- Phase 11 (planning): calcForward adapted for multi-gateway (accepts resolved rate params, not plan objects)
- Phase 11 (planning): calcForward adds CPT step (Tiendanube transaction cost) not in original prototype
- Phase 11 (planning): Binary search uses epsilon convergence (not fixed 100 iterations), upper bound = max(cost*20, 100000)
- Phase 11 (planning): CalculadoraModule imports TiendanubeConfigModule + CostsModule + ProductsModule
- Phase 11 (pre-mortem fix): product.currentPrice is STRING from TypeORM decimal — must parseFloat() in calcBatch before passing to calcForward
- Phase 11 (pre-mortem fix): rate.gateway from getAll() raw SQL is plain JSON with snake_case keys — use .slug for matching (same in both cases)
- Phase 11 (pre-mortem fix iter3): rate.payment_method and rate.withdrawal_days are ALSO snake_case at runtime — resolveRates MUST use rate.payment_method and rate.withdrawal_days (not camelCase) **SUPERSEDED by 10-03 gap closure fix**
- Phase 11 (pre-mortem fix iter3): Use toast.error() from sonner for inverse error display (Shadcn Alert not installed)
- Phase 11 (pre-mortem fix iter3): CalculadoraClient.tsx MUST have 'use client' directive (useState/useSession/useCallback)
- Phase 11 (pre-mortem fix iter3): ModeToggle uses Shadcn Tabs (installed), NOT ToggleGroup (not installed)
- Phase 11 (pre-mortem fix iter3): CalculadoraService constructor injects ProductsService (needed for calcBatch findAll)
- Phase 11 (pre-mortem fix iter4): ratePercent is NaN on ALL gateway AND installment rate objects from getAll() — raw SQL returns rate_percent (snake_case), parseFloat(rate.ratePercent) = parseFloat(undefined) = NaN. resolveRates MUST use (rate as any)['rate_percent'] and parseFloat it manually. **SUPERSEDED by 10-03 gap closure fix — SQL now returns camelCase aliases, workarounds removed**
- Phase 11 (pre-mortem fix iter4): IVA/IIBB stored as percentages (21, 3.5) but formulas need fractions (0.21, 0.035) — resolveRates divides by 100. Gateway/installment/CPT rates stay as percentages (divided by 100 inline in formula).
- Phase 11 (pre-mortem fix iter4): Controller returns CalcResult directly — ResponseInterceptor wraps to { data: CalcResult }. Do NOT return { data: CalcResult } manually.
- Phase 11 (pre-mortem fix iter4): taxConfig can be null from getAll() — resolveRates throws NotFoundException if null
- Phase 11 (frontend): GatewaySelectors uses derived state (useMemo for effectiveMethod/effectiveDays) instead of setState-in-useEffect for cascading resets (React lint compliance)
- Phase 11 (frontend): Sidebar "Herramientas" group with Calculadora link visible to ALL users (outside isAdmin conditional)
- Phase 11 (gap closure): Infinite re-render caused by unstable onConfigChange prop + notifyChange useCallback depending on it. Fix: useCallback in parent + useRef for callback in child.
- [Phase 12]: Scenario overrides use full-replace strategy (delete+insert in QueryRunner transaction)
- [Phase 12]: calculate returns both simResult and realResult per product for override vs real comparison
- [Phase 12]: DebouncedInput uses uncontrolled defaultValue + React key remount instead of useEffect+setState (ESLint compliance)
- [Phase 12]: GatewayPlanSelector uses useRef for parent onChange callback to prevent re-render loops (Phase 11 fix pattern)
- [Phase 12]: handleSaveAndCalculate uses Promise.all for parallel PUT overrides + PUT config, then sequential GET calculate
- [Phase 12]: Escenarios visible to all users (admin + investor), not added to ADMIN_ONLY_ROUTES
- [Phase 12 gap closure]: Admin delete bypasses owner filter via role param in remove(), canDelete prop separates delete visibility from ownership in ScenarioCard
- [Phase 12.1]: TiendanubeConfig GET endpoints have NO @RequirePermission -- accessible to any authenticated user for calculator-only users
- [Phase 12.1]: GET endpoints protected with can_view_* permissions for defense-in-depth (not just mutations)
- [Phase 12.1]: Frontend Permissions type mirrors backend Role entity (11 boolean flags)
- [Phase 12.1]: Stale JWT detection: if token has role (string) but no permissions, clears backendToken to force re-auth
- [Phase 12.1]: ROUTE_PERMISSIONS map replaces ADMIN_ONLY_ROUTES for per-route permission checks (10 routes)
- [Phase 12.1]: canEdit prop pattern replaces isAdmin across all 14 client components
- [Phase 12.1]: AppSidebar groups conditionally rendered based on specific permission flags
- [Phase 12.1]: UsersClient interim fix: role typed as string | object union with typeof guard (Plan 04 replaces fully)
=======
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
- 05-01: SKU format: type.name.finish.color.size using skuCode SMALLINT from each dimension
- 05-01: Partial unique index on products.sku_code WHERE is_active = true (same pattern as supplies)
- 05-01: Batch creation via entityManager.transaction with cartesian product of colors x sizes
- 05-01: UpdateProductDto requires all 5 dimension IDs (SKU regenerates on any change)
- 05-01: ProductsService exported for future BOM and cost calculation modules
- 05-01: product_sizes "Unico" gets sku_code=0 (sentinel for single-size products)
- 05-02: BOM version swap: deactivate old + insert new in entityManager.transaction
- 05-02: ProductPriceHistory append-only with DISTINCT ON for current price in findAll
- 05-02: Supply deactivation guard checks bomRepo.count before allowing is_active=false
- 05-02: Batch routes defined BEFORE :id routes to avoid NestJS UUID param collision
- 05-02: lint-staged: prettier --write instead of --check for reliable Windows commit hooks
- 05-03: Batch create uses 4-step wizard dialog matching cartesian product backend pattern
- 05-03: SKU preview in edit dialog updates live via useMemo on watched form values
- 05-03: BOM fetched lazily on row expand (not preloaded) to avoid N+1 on page load
- 05-03: BOM edit, price, and history buttons stubbed for plan 05-04 wiring
- 05-04: BomGroupEditorDialog uses majority BOM detection via serialized JSON comparison
- 05-04: Migration fix: dynamic ROW_NUMBER fallback for user-created catalog items not in CASE statements
- 05-04: Auth ClientFetchError is pre-existing (not phase 5), documented but not fixed
- 06-01: CostsModule imports only entity repos (no ProductsModule/SuppliesModule) to avoid circular deps
- 06-01: Cost breakdown excluded from findAll list response (null) for performance, included only in findOne detail
- 06-01: findOneWithPrice added to ProductsService for single-product price fetch in detail view
- 06-02: Expanded row fetches product detail in parallel with BOM to get costBreakdown (list returns null)
- 06-02: formatMargin calculates markup % (margin/cost) not gross margin (margin/price)
- 06-02: Product name Link with stopPropagation to avoid row expand on click
- [Phase 12.1]: Permission flags as 11 booleans on Role entity for query simplicity
- [Phase 12.1]: PermissionsGuard replaces RolesGuard as APP_GUARD; old guard deprecated for Plan 02
- [Phase 12.1]: TypeOrmModule.forFeature([User, Role]) in UsersModule avoids circular dependency
>>>>>>> worktree-agent-a70315cb

### Pending Todos

10 todos pending. See `.planning/todos/pending/`.
Latest: Fix JWT expiry silent failure - redirect to login on 401 (auth)

Todos absorbed into Phase 8 plans:

- proxy.ts runtime verification → 08-01 Task 1 (renamed to middleware.ts)
- ExpenseFormDialog category Select bug → 08-02 Task 2 (fix with watch())
- Delete deactivateBySupplier dead code → 08-01 Task 2
- Role-based redirect on login → 08-01 Task 1 (USER redirected to / by middleware)

### Blockers/Concerns

- proxy.ts vs middleware.ts naming RESOLVED: proxy.ts export `proxy` is NOT what Next.js 16 needs. The file must be named middleware.ts with export named `middleware`. Previous research was partially incorrect — runtime verification confirmed it was not firing.
- deactivateBySupplier RESOLVED: deleted as dead code in 08-01
- SIRTAC/IIBB aliquot: single configurable value, admin enters manually. Not automated. (Phase 10 — addressed in plan)
- Raw SQL snake_case bug: DIAGNOSED in Phase 10 UAT, gap closure plan 10-03 created. Root cause = getLatestGatewayRates() and getInstallmentRates() use SELECT * which returns PostgreSQL snake_case column names, but parse helpers expect camelCase. Fix = column aliases in SQL.
- Calculadora infinite re-render: DIAGNOSED in Phase 11 UAT, gap closure plan 11-03 created. Root cause = handleGatewayConfigChange not wrapped in useCallback + notifyChange/useEffect circular dependency via onConfigChange prop. Fix = useCallback in parent + ref-based callback pattern in child.

### Known Issues

- Auth ClientFetchError on frontend homepage: NextAuth session endpoint returns HTML instead of JSON. Pre-existing from v1.0.

## Session Continuity

<<<<<<< HEAD
Last session: 2026-03-30T16:06:00Z
Stopped at: Completed 12.1-03-PLAN.md (frontend permission system)
Resume file: .planning/phases/12.1-dynamic-roles-with-configurable-permissions/12.1-03-SUMMARY.md
=======
Last session: 2026-03-30T15:55:49.837Z
Stopped at: Completed 12.1-01-PLAN.md (Backend Permission System)
Resume file: None
>>>>>>> worktree-agent-a70315cb
