# Project Research Summary

**Project:** Nemea v1.1 — Tiendanube Pricing Simulation & Investor Dashboard
**Domain:** E-commerce pricing tool + investor margin analysis for artisan leather goods (Argentine market)
**Researched:** 2026-03-26
**Confidence:** HIGH

## Executive Summary

Nemea v1.1 extends a fully-deployed NestJS + Next.js cost management app with three capability clusters: Tiendanube pricing simulation (forward/inverse calculadora), an investor margin dashboard with scenario modeling, and foundational hardening of auth and code quality. All research confirms that zero new npm dependencies are required — the existing stack (NestJS 11, TypeORM, Next.js 16, Shadcn/ui, Zod) covers every feature. The primary additions are architectural: four new NestJS modules (TiendanubeConfigModule, CalculadoraModule, ScenariosModule, DashboardModule) and four new frontend pages. The calculadora logic is already proven in a working prototype; migration means connecting it to DB-stored config, not inventing new math.

The recommended approach is strict build-order sequencing driven by a clean dependency DAG: hardening first (auth 401 handling, role enforcement, DRY cleanup), then Tiendanube config tables, then the calculator, then scenarios, then the dashboard. This order is non-negotiable — every layer depends on the one below. A critical architectural decision is to keep the 14-step forward calculation as a pure backend service (`CalculadoraService`) and never duplicate it in the frontend, ensuring the dashboard and the calculadora page always produce identical numbers from a single source of truth.

The highest-risk areas are floating-point precision in the chained percentage calculations (mitigated by rounding only at the end of the full calculation chain, not at intermediate steps), scenario data isolation (prevented by strict use of separate `scenario_overrides` table, never touching `product_price_history`), and missing role enforcement on existing backend endpoints (requires adding `@Roles(Role.ADMIN)` to all existing controllers before investors access the app). Auth hardening must be the first phase — broken JWT expiry handling will degrade every subsequent feature's user experience.

---

## Key Findings

### Recommended Stack

**Zero new dependencies needed.** All v1.1 features are achievable with the current stack. For financial calculations, native JavaScript `Number` with `Math.round(value * 100) / 100` applied once at the end of the full chain is sufficient for ARS 2-decimal amounts — `decimal.js` is explicitly rejected as overkill. TypeORM returns `decimal` columns as strings; the conversion boundary (TypeORM entity to service) must be consistent: all service interfaces accept `number`, conversion happens once at the repo boundary.

**Core technologies (existing, validated for v1.1):**
- NestJS 11 + TypeORM: four new modules (TiendanubeConfig, Calculadora, Scenarios, Dashboard) follow identical patterns to existing CatalogsModule and ExpensesModule
- Next.js 16: `proxy.ts` is correctly named and located for Next.js 16 (not a naming bug); the real auth gap is 401 handling in `apiClientFetch`
- PostgreSQL: 5 new tables added via TypeORM migrations; all extend `BaseEntity` (UUID PK, timestamps)
- Shadcn/ui + Radix UI: all new UI components (tables, cards, collapsible groups, selects) use already-installed primitives
- Zod + react-hook-form: calculadora form and scenario editor follow existing patterns exactly

**Explicitly rejected additions:** `decimal.js`, `recharts` / `chart.js`, `@tanstack/react-table`, Redis / caching, background job queues, i18n libraries, a separate pricing microservice.

### Expected Features

**Must have (table stakes — v1.1 milestone fails without these):**
- Auth hardening: 401 redirect in `apiClientFetch` + role-based redirects in `proxy.ts`
- Role enforcement: `@Roles(Role.ADMIN)` on ALL existing controllers (products, supplies, suppliers, catalogs, expenses)
- DRY cleanup: extract shared `SupplyOption` (defined 8x), `UNIT_LABELS` (5x), `formatDate` (4x) before adding new features
- Users admin page: frontend UI for existing `/users` CRUD endpoints (backend already complete)
- Tiendanube config tables: admin-editable plans (4), installment rates (5 tiers), tax config (IVA + IIBB) — all rates verified against official Tiendanube docs March 2026
- Calculadora forward mode: price to profit after all Tiendanube deductions (14-step formula, proven in prototype)
- Calculadora inverse mode: desired profit to required selling price (binary search over forward)
- Investor dashboard: all products with cost, selling price, and gross margin (read-only)
- Scenario simulator: named what-if scenarios with per-product price overrides, user-scoped

**Should have (differentiators, ship if time allows):**
- Net margin after Tiendanube deductions column in dashboard (the "killer column" — real profit vs gross margin)
- Product hierarchical grouping: type > name > finish tree (frontend-only, no backend changes)
- Calculadora auto-populated product cost from DB (eliminates manual cost entry, UX improvement)
- BOM group editor scoped to product name instead of type (correctness fix for real business model)
- `lastVerifiedAt` timestamp on Tiendanube config + link to official rates page

**Defer to v1.2+:**
- Historical margin tracking (needs time-series data accumulation first)
- PDF export (browser print-to-PDF is sufficient for 1-2 investors)
- Multi-scenario side-by-side comparison (one-at-a-time is adequate for the user count)
- Tiendanube API integration (no public rates API, massive scope for unclear gain)

### Architecture Approach

Four new NestJS modules slot cleanly into the existing dependency graph with zero circular dependencies. The DAG is strictly one-directional: `TiendanubeConfigModule` (no deps) feeds `CalculadoraModule` (depends on CostsModule + TiendanubeConfigModule), which feeds `ScenariosModule`, which feeds `DashboardModule` (aggregates everything). The `CostsModule` is the linchpin — it avoids importing other feature modules by using `TypeOrmModule.forFeature()` directly, preserving its clean base-layer position. `DashboardModule` is a consumer-only aggregation layer that exports nothing, preventing any return dependencies.

**Major components:**
1. `TiendanubeConfigModule` — CRUD for plan rates, installment rates, tax config. Seed data via migration. Admin-only write, any-auth read. 3 entities, standalone.
2. `CalculadoraModule` — Stateless `CalculadoraService` implementing `calcForward()` (14 steps), `calcInverse()` (binary search with epsilon convergence), `calcBatch()` (for dashboard). Pure math, no DB writes.
3. `ScenariosModule` — User-scoped CRUD for named scenarios + price override tables. Strict isolation: never writes to `product_price_history`. 2 entities.
4. `DashboardModule` — `DashboardService` composes data from all modules in 5 queries total + in-memory O(n) calculation loop. Consumer-only, exports nothing.
5. Frontend: 4 new pages (`/calculadora`, `/dashboard`, `/configuracion/tiendanube`, `/admin/usuarios`) + shared types under `@/types/` + shared formatters under `@/lib/formatters.ts`.

### Critical Pitfalls

1. **Floating-point accumulation across 7 intermediate calculation steps** — Round only ONCE at the end of the full `calcForward()` chain, not at each intermediate step. Test: forward at $87,000 then inverse targeting result must round-trip within $0.01.
2. **Scenario data leaking into real price history** — `ScenariosService` must NEVER write to `product_price_history`. Enforce at the module level by only registering `Scenario` + `ScenarioOverride` entity repos. Detect: query `product_price_history` after scenario creation — zero new records expected.
3. **Missing role enforcement on existing endpoints** — All current controllers lack `@Roles(Role.ADMIN)`. A USER-role investor can navigate to `/productos` and see all cost data. Fix this in the hardening phase before investors access the system.
4. **Silent 401 on JWT expiry** — `apiClientFetch` throws a generic error on 401; investors see broken UI with no recovery path. Add `window.location.href = '/login'` on 401. Handle 403 separately ("no permissions" message, not login redirect).
5. **Dashboard N+1 via naive service composition** — `DashboardService` must load Tiendanube config ONCE, not inside a per-product loop. Target: no more than 6 SQL queries per dashboard request regardless of product count.

---

## Implications for Roadmap

Based on the dependency DAG and pitfall analysis, the optimal phase structure is:

### Phase 1: Hardening
**Rationale:** Auth is broken (no 401 handling, no role enforcement), and 8 duplicate types slow all subsequent work. Missing role enforcement is a security gap that must be closed before investors access any endpoint. DRY cleanup reduces friction for every subsequent feature phase. `deactivateBySupplier` dead code should be resolved here.
**Delivers:** Secure, investment-grade foundation with all existing controllers protected. Shared type system established. Users admin page live. JWT expiry handled gracefully.
**Addresses features:** Auth hardening (middleware + 401), role enforcement on all existing endpoints, DRY cleanup, users admin page.
**Avoids pitfalls:** Pitfall 2 (JWT expiry silent failure), Pitfall 8 (role bypass on existing routes), Pitfall 5 (DRY cleanup breaking contracts — diff types before extracting), dead code.

### Phase 2: Tiendanube Config
**Rationale:** This is the single dependency gate for both the calculadora and the net margin dashboard column. Nothing in the pricing simulation stack can be built before rates are in the DB. Building it as a standalone module also validates the new entity/migration pattern before higher-complexity modules.
**Delivers:** Admin-editable rate tables (4 plans, 5 installment tiers, IVA + IIBB). Config CRUD frontend page with "verify rates" link. Seed data with March 2026 verified rates.
**Implements:** `TiendanubeConfigModule` — foundation of the new dependency DAG.
**Avoids pitfalls:** Pitfall 7 (stale hardcoded rates — rates live in DB from day one), Pitfall 10 (decimal string convention established here: parse at repo boundary, number in service interfaces).

### Phase 3: Calculadora
**Rationale:** Depends on Phase 2 (config) and the existing CostsModule. The pure calculation service is a prerequisite for both the calculadora page AND the dashboard net margin column AND scenario calculations. Proving the formula migration here prevents duplication later.
**Delivers:** `POST /calculadora/forward` and `POST /calculadora/inverse` endpoints. Frontend calculadora page with product cost auto-population from DB. Binary search with epsilon convergence and bounds validation.
**Uses:** `CalculadoraService` as pure math, `CostsService.calculateForProduct()` for DB lookup, `TiendanubeConfigService` for rates.
**Avoids pitfalls:** Pitfall 1 (floating point — round only at chain end), Pitfall 6 (inverse edge cases — epsilon convergence, unreachable-target error return, zero-cost product guard).

### Phase 4: Scenarios
**Rationale:** Scenarios are a layer on top of the calculadora. They require `CalculadoraService.calcBatch()` which exists after Phase 3. This is the most complex new feature and benefits from all its dependencies being proven before the integration layer is built.
**Delivers:** Named what-if scenarios, per-product price overrides, user-scoped visibility. `PUT /scenarios/:id/overrides` bulk upsert. `GET /scenarios/:id/calculate` endpoint for full margin recalculation with overrides.
**Avoids pitfalls:** Pitfall 4 (scenario data isolation — strict separate tables, never touch real pricing data), Pitfall 11 (scope creep — one active scenario vs reality, no side-by-side in v1.1).

### Phase 5: Investor Dashboard
**Rationale:** Last because it consumes all other modules. The `DashboardService` composition (products + costs + calculadora + scenarios) is the final integration. Building it last means all its dependencies are independently tested first.
**Delivers:** `/dashboard` page with catalog margin summary, net margin after Tiendanube deductions, scenario selector, product hierarchy grouping, aggregated KPI cards.
**Avoids pitfalls:** Pitfall 3 (aggregation performance — 5 queries total, in-memory loop), Pitfall 4 (scenario isolation through read-only service composition).

### Phase 6: Product UX (parallel candidate)
**Rationale:** Fully independent of Phases 2-5 — no backend changes, no new entities. Can be parallelized with any other phase or deferred. Hierarchical product table + BOM group editor scoping are frontend-only refactors.
**Delivers:** Type > Name > Finish collapsible product tree using existing Radix Collapsible. BOM group editor scoped to product name instead of type.

### Phase Ordering Rationale

- **Hardening before everything:** Role enforcement is a security requirement. The 401 handling gap degrades the experience of every subsequent feature. DRY cleanup prevents type inconsistencies from compounding across new features.
- **Config before Calculator:** Hard dependency — the calculator reads rates from DB on every call. Building the calculator against hardcoded rates would require a second migration step.
- **Calculator before Scenarios:** `ScenariosModule` imports `CalculadoraModule.calcBatch()`. This dependency cannot exist until Phase 3 is complete.
- **Scenarios before Dashboard:** `DashboardService` composes `ScenariosService.findOne()` for scenario-aware views.
- **Product UX is parallel:** No dependencies on any v1.1 module. Can start after Phase 1 DRY cleanup establishes shared types.

### Research Flags

Phases likely needing `/gsd:research-phase` during planning:
- **Phase 3 (Calculadora):** The number representation decision (round at chain end vs integer centavos) should be validated against the $87,000 prototype test case before writing the service. The binary search edge case matrix (negative profit, zero-cost product, unreachable target) needs explicit test design before implementation.
- **Phase 4 (Scenarios):** The API shape for `PUT /scenarios/:id/overrides` (bulk upsert semantics, replace-all vs partial update) has no reference in the existing codebase. Design the contract before the controller is written.

Phases with well-documented patterns (skip research-phase):
- **Phase 1 (Hardening):** All tasks are targeted fixes to known issues. `@Roles` pattern, `apiClientFetch` 401 handling, and DRY extraction are all well-understood changes.
- **Phase 2 (Tiendanube Config):** Pure CRUD module following identical pattern to `CatalogsModule`. Seed data is verified. No unknowns.
- **Phase 5 (Dashboard):** The `DashboardService` composition pattern is fully specified in ARCHITECTURE.md. No novel patterns.
- **Phase 6 (Product UX):** Frontend-only Radix Collapsible grouping. No unknowns.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Zero new libraries. All decisions verified against existing codebase and official docs. Version compatibility table confirmed. |
| Features | HIGH | Core features drawn from working prototype + verified Tiendanube rates (official help center, March 2026). Scenario UX patterns are MEDIUM (FP&A tools adapted for 2-user leather goods app). |
| Architecture | HIGH | Dependency DAG verified against actual module wiring in codebase source files. No circular dependency risk. All new module patterns match existing modules exactly. |
| Pitfalls | HIGH | Grounded in actual code analysis from v1 senior review, official docs (TypeORM issue #2937, Next.js 16 proxy docs), and confirmed runtime behavior. |

**Overall confidence:** HIGH

### Gaps to Address

- **Calculadora number representation conflict:** STACK.md recommends `Math.round(x * 100) / 100` matching existing `CostsService`. PITFALLS.md recommends rounding only at chain end (same conclusion) but also mentions integer centavos as an alternative. Resolve at Phase 3 start: run the 14-step formula both ways against the $87,000 test case and measure discrepancy. If sub-centavo, `Math.round` at chain end is sufficient.
- **proxy.ts vs middleware.ts naming:** STACK.md says `proxy.ts` is correctly structured for Next.js 16. ARCHITECTURE.md says rename to `middleware.ts`. This is a direct conflict. Requires runtime verification at Phase 1 start — check Next.js 16 changelog to confirm the correct filename for route interception.
- **SIRTAC / IIBB aliquot:** Rate varies per vendor based on COMARB padron (up to 5%). The app stores a single configurable value — explicitly scoped as "good enough estimation." Admin must look up their actual aliquot and enter it. No automation is possible or required.
- **`deactivateBySupplier` wiring decision:** Either wire it into `SuppliersService.toggleStatus()` or delete it. Requires a business logic call from the user before Phase 1 execution.

---

## Sources

### Primary (HIGH confidence)
- Nemea codebase: `nemea-back/src/costs/costs.module.ts`, `nemea-back/src/products/products.module.ts`, `nemea-back/src/app.module.ts` — actual module wiring verified
- Nemea codebase: `nemea-front/src/proxy.ts`, `nemea-front/src/lib/api-client.ts` — auth gap analysis
- `_archive/calculadora/calculadora-tiendanube.jsx` — proven forward/inverse formula with business test cases
- `.claude/docs/planning/business-rules.md` — Tiendanube rates, `calcForward` steps, test case ($87,000 wallet)
- [Tiendanube comisiones oficiales](https://ayuda.tiendanube.com/es_AR/pago-nube-2/cuales-son-las-comisiones-de-pago-nube) — plan rates verified March 2026
- [Tiendanube SIRTAC / IIBB](https://ayuda.tiendanube.com/es_AR/pago-nube-2/que-costos-debo-tener-en-cuenta-al-vender-con-pago-nube) — retention regime
- [SIRTAC Argentina](https://www.argentina.gob.ar/economia/politicatributaria/armonizacion/sirtac) — official IIBB retention system

### Secondary (MEDIUM confidence)
- [Auth.js v5 session protection](https://authjs.dev/getting-started/session-management/protecting) — proxy.ts patterns
- [TypeORM decimal as string — Issue #2937](https://github.com/typeorm/typeorm/issues/2937) — decimal column behavior confirmed
- [How to properly handle decimals with TypeORM](https://medium.com/@matthew.bajorek/how-to-properly-handle-decimals-with-typeorm-f0eb2b79ca9c) — column transformer pattern
- [Currency calculations in JavaScript — Honeybadger](https://www.honeybadger.io/blog/currency-money-calculations-in-javascript/) — integer centavos pattern
- [Understanding N+1 in TypeORM](https://medium.com/@bloodturtle/understanding-the-n-1-problem-in-typeorm-b93757dca974) — dashboard query optimization
- Craftybase, Cube, Finmark, Runway — scenario simulator and investor dashboard UX patterns

### Tertiary (LOW confidence)
- [Shadcn tree view component](https://github.com/MrLightful/shadcn-tree-view) — considered and rejected in favor of Radix Collapsible for product hierarchy

---
*Research completed: 2026-03-26*
*Ready for roadmap: yes*
