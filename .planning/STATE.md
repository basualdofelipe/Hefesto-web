---
gsd_state_version: 1.0
milestone: v1.1
milestone_name: Tiendanube & Investor Dashboard
status: executing
stopped_at: Phase 12.5 context gathered
last_updated: "2026-05-26T20:22:11.420Z"
last_activity: 2026-05-26
progress:
  total_phases: 18
  completed_phases: 15
  total_plans: 48
  completed_plans: 48
  percent: 83
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-26)

**Core value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automaticamente cuando cambian los precios de los insumos.
**Current focus:** Phase 12.5 — demo-login-and-code-readiness-for-portfolio

## Current Position

Phase: 12.4 (user-management-edit-user-role-delete-user-admin-self-lockou) — EXECUTING
Plan: 2 of 8
Status: Ready to execute
Last activity: 2026-05-26

Progress (v1.1): [####################....] 83% (5/6 phases)
Progress (overall): [#########################.] 96% (12/13 phases)

## Performance Metrics

**Velocity (from v1.0):**

- Total plans completed: 28
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
| 12.1 | 4 | - | - |
| 12.2 | 3 | - | - |

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
| Phase 12.1 P01 | 35 | 2 tasks | 27 files |
| Phase 12.1 P02 | 25min | 2 tasks | 13 files |
| Phase 12.1 P03 | 40min | 2 tasks | 29 files |
| Phase 12.1 P04 | 18min | 2 tasks | 5 files |
| Phase 12.1 P04 | 18min | 3 tasks | 6 files |
| Phase 12.2 P01 | 12min | 2 tasks | 3 files |
| Phase 12.2 P02 | 4 | 2 tasks | 8 files |
| Phase 12.2 P03 | 4min | 1 tasks | 23 files |
| Phase 12.4 P01 | 2.5min | 2 tasks | 2 files |
| Phase 12.4 P02 | 4min | 2 tasks | 2 files |
| Phase 12.4 P03 | 18min | 2 tasks | 3 files |
| Phase 12.4 P04 | 4min | 2 tasks | 2 files |
| Phase 12.4 P05 | 8min | 3 tasks | 3 files |
| Phase 12.4 P06 | 4.25min | 2 tasks | 2 files |
| Phase 12.4 P07 | ~3min | 3 tasks (1+2 PASS, 3 SKIP→G-1) | 1 file (SUMMARY only) |
| Phase 12.4 P08 | 7min | 3 tasks | 5 files |

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: TiendanubeConfigModule named explicitly to avoid collision with @nestjs/config's ConfigModule
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
- [Phase 12.1]: Permission flags as booleans on Role entity (11 flags), not a separate permissions table
- [Phase 12.1]: Backend re-reads permissions from DB on every request via eager-loaded role (JWT carries permissions only for frontend)
- [Phase 12.1]: UsersModule uses TypeOrmModule.forFeature([User, Role]) directly to avoid circular dep with RolesModule
- [Phase 12.1]: RolesGuard retained with backward-compat logic (canManageUsers=ADMIN) until Plan 02 migrates all controllers
- [Phase 12.1]: TiendanubeConfig GET endpoints left permission-free so calculator-only users can access TN config data without can_view_products
- [Phase 12.1]: ScenariosService.remove uses permissions.canManageUsers as admin bypass — semantic alignment with user management = admin-level access
- [Phase 12.1]: All GET endpoints require view permissions (defense in depth) — no unguarded data reads even with valid JWT
- [Phase 12.1]: Stale JWT detection via key deletion (token.role exists + token.permissions absent) clears backendToken to force re-auth without rotating NEXTAUTH_SECRET
- [Phase 12.1]: AppSidebar renders nav groups conditionally per permission — each group only shown when user has relevant flag
- [Phase 12.1]: UsersClient interim role union type (object|string) with typeof guard — Plan 04 replaces with full RoleOption type
- [Phase 12.1]: z.boolean() without .default() used in roleSchema — avoids optional inference that breaks zodResolver type compatibility
- [Phase 12.1]: Shared RoleRow and RoleOption types in nemea-front/src/types/role.ts — eliminates Plan 03 interim union type and prevents duplication between /roles and /usuarios pages
- [Phase 12.1]: z.boolean() without .default() in roleSchema — avoids optional inference that breaks zodResolver type compatibility
- [Phase 12.1]: Circular import between Role and User entities fixed via import type + TypeORM string-form entity reference in @OneToMany decorator
- [Phase 12.2]: D-09: sort_order is SMALLINT on product_sizes table, database-driven ordering not frontend sorting
- [Phase 12.2]: Used productSizeRepo directly in CatalogsService special case to avoid any cast (ESLint/CLAUDE.md compliance)
- [Phase 12.2]: D-04 (RSC) takes precedence over D-03 hook detail: home page uses await auth() instead of usePermissions() — RSC cannot use React hooks
- [Phase 12.2]: D-01: only Name and Finish collapse default changed to false — TypeGroup stays true (deliberately)
- [Phase 12.2]: D-08 applied: font-medium (500) → font-semibold (600) for all emphasis contexts; font-bold (700) → font-semibold (600) for stat numbers — 2-weight system fully implemented
- [Phase ?]: [Phase 12.4 P01]: Controller stays pure delegate — guards live in UsersService (D-04). UpdateUserDto declared as new class (not PartialType) to avoid inheriting email + @IsNotEmpty constraints from CreateUserDto.
- [Phase ?]: [Phase 12.4 P01]: @HttpCode(204) explicit on DELETE — NestJS defaults to 200 with no body; required for REQ-12.4-3 contract.
- [Phase ?]: [Phase 12.4 P02]: Arrow function () => raw SQL is the only TypeORM-supported way to mix raw SQL with parametrization (issue #1580). user: { id } object-form coexists with arrow-function name in .set() — no fallback needed.
- [Phase ?]: [Phase 12.4 P02]: transferOwnership specs assert QueryBuilder chain shape but NOT the literal SQL string — implementation detail. Local beforeEach restores mockReturnThis() after outer jest.clearAllMocks() to preserve chain semantics.
- [Phase ?]: [Phase 12.4 P03]: EditUserDialog uses RHF values API (not defaultValues + useEffect/reset) — canonical pattern for prop-driven prefill, also fixes the re-open-with-different-user case where the original pattern desynced watch() from Radix Select state.
- [Phase ?]: [Phase 12.4 P03]: jest.setup.ts polyfills ResizeObserver / scrollIntoView / setPointerCapture — required by Radix primitives under jsdom. Environment shims, not behavior mocks. Reusable by all future Radix-based component tests.
- [Phase ?]: [Phase 12.4 P03]: Use fireEvent.click on submit button (not userEvent.click) in Radix Dialog tests — Radix sets body pointer-events:none on open which user-event respects even with pointerEventsCheck:Never.
- [Phase ?]: [Phase 12.4 P03]: Mock UUIDs in tests MUST be valid v4 — zod .uuid() rejects literals like 'role-uuid'. Use VICTIM_ID/EDITOR_ROLE_ID/ADMIN_ROLE_ID constants at top of test files.
- [Phase ?]: [Phase 12.4 P04]: DeleteUserAlertDialog uses className override (not variant=destructive) on AlertDialogAction — matches RolesClient/ScenarioListClient project convention
- [Phase ?]: [Phase 12.4 P04]: AlertDialog stays open on backend error (parent controls close via onOpenChange) — gives user retry-or-cancel choice with Spanish backend message visible
- [Phase ?]: [Phase 12.4 P05]: Field-by-field assignment instead of repo.merge — defense-in-depth even with global ValidationPipe whitelist (T-12.4-22)
- [Phase ?]: [Phase 12.4 P05]: Last-admin guard computed BEFORE write (TypeScript logic on post-change state) — simpler than transaction re-count and matches roles.service.ts pattern
- [Phase ?]: [Phase 12.4 P05]: transferOwnership ordered BEFORE manager.delete inside QueryRunner tx — UPDATE moves rows off victim FK before DELETE looks, so ON DELETE CASCADE never fires (RESEARCH Finding 10)
- [Phase ?]: [Phase 12.4 P05]: Fast-fail guards (self-delete + 404 + last-admin) BEFORE opening QueryRunner — avoids orphan transactions on rejection paths
- [Phase ?]: [Phase 12.4 P06]: UsersClient integration complete — Editar+Borrar buttons per row, (tu cuenta) marker on own row (D-09), Editar always visible with internal disable via isOwnRow prop (D-08). Controller spec rewrite closes the final 5 tsc errors from Plan 01; backend tsc --noEmit fully clean.
- [Phase ?]: [Phase 12.4 P07]: Phase verification gate PASS — backend triple-gate (tsc 0, lint 0, jest 107/107) + frontend triple-gate (tsc 0, lint 0, jest 10/10 on EditUserDialog + DeleteUserAlertDialog suites). Pre-existing NextAuth v5 ESM Jest parse fail on page.test.tsx is heritage from Phase 03/12.1, NOT a 12.4 regression (confirmed via git stash round-trip in Plan 03). Manual 8-AC smoke (Task 3) explicitly skipped per user decision (Windows + single Google account impracticable for 2-admin simultaneous login) and registered as Gap G-1 for /gsd:verify-work — full unit-test coverage map documented (AC3/AC4/AC7/AC8 + rollback covered by users.service.spec 24/24; AC1/AC2/AC5/AC6 require real-DB smoke).
- [Phase ?]: [Phase 12.4 P08]: countOtherActiveAdmins — FOR UPDATE + COUNT is illegal in PostgreSQL; removed setLock. SERIALIZABLE isolation on the outer transaction already prevents concurrent race conditions without a row-lock on the aggregate.
- [Phase ?]: [Phase 12.4 P08]: transferOwnership uses 'updatedAt' (entity property name) not 'updated_at' (DB column) in TypeORM QueryBuilder .set() — TypeORM resolves .set() keys by property name and throws EntityPropertyNotFoundError when given column name.

### Roadmap Evolution

- Phase 12.4 inserted after Phase 12: User management — edit user role, delete user, admin self-lockout guard (URGENT) — surfaced by 12.1 UAT: custom roles undeleteable once assigned because /usuarios lacks edit-role and delete-user flows. Closes pending todos `edit-user-role.md` and `delete-user.md`.
- Phase 12.5 inserted after Phase 12: Demo login and code-readiness for portfolio (URGENT)

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

Last session: 2026-05-26T20:22:11.406Z
Stopped at: Phase 12.5 context gathered
Resume file: .planning/phases/12.5-demo-login-and-code-readiness-for-portfolio/12.5-CONTEXT.md
