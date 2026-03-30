# Roadmap: Nemea

## Overview

Nemea is a cost management and pricing simulation system for an artisan leather goods business.
v1.0 replaced Google Sheets with a production-grade NestJS + Next.js app covering products, supplies,
costs, and expenses. v1.1 hardens the foundation, adds Tiendanube pricing simulation (forward/inverse
calculadora with real costs), and delivers an investor dashboard with scenario modeling for margin analysis.

## Milestones

- v1.0 MVP - Phases 1-7 (shipped 2026-03-11)
- 🚧 **v1.1 Tiendanube & Investor Dashboard** - Phases 8-13 (in progress)

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

### v1.0 MVP (Phases 1-7) - SHIPPED 2026-03-11

- [x] **Phase 1: Backend Scaffold** - NestJS 11 app running with TypeORM migrations, Docker Compose, Swagger
- [x] **Phase 2: Auth** - Google OAuth via NextAuth exchanges for NestJS JWT, global guard, email whitelist, roles
- [x] **Phase 3: Catalogs and Suppliers** - CRUD for all 5 product dimensions, supply types, and suppliers
- [x] **Phase 4: Supplies and Price History** - Supply CRUD with append-only price history and composite index
- [x] **Phase 5: Products and BOM** - Product CRUD with SKU, material composition with version history, selling price
- [x] **Phase 6: Cost Calculation** - Dynamic cost per product (batched DISTINCT ON query), visible in product list and detail
- [x] **Phase 7: Expenses** - Expense tracking with editable categories, grouped by month

### v1.1 Tiendanube & Investor Dashboard (Phases 8-13)

- [x] **Phase 8: Hardening** - Auth middleware, 401 handling, role enforcement, users admin, DRY cleanup, produccion externa
- [x] **Phase 9: Product UX** - Hierarchical product grouping (type > name > finish), BOM group editor scoped to name level
- [ ] **Phase 10: Tiendanube Config** - Admin-editable rate tables for plans, installments, and taxes
- [ ] **Phase 11: Calculadora** - Forward (price to profit) and inverse (profit to price) with real costs and Tiendanube deductions
- [ ] **Phase 12: Scenarios** - User-scoped what-if scenarios with price overrides and recalculated margins
- [ ] **Phase 13: Investor Dashboard** - Catalog summary with margins, Tiendanube net profit, scenario selector, aggregate KPIs

## Phase Details

<details>
<summary>v1.0 MVP (Phases 1-7) - SHIPPED 2026-03-11</summary>

### Phase 1: Backend Scaffold
**Goal**: NestJS 11 backend runs locally and on Railway with migrations wired up before any entity is written
**Depends on**: Nothing (first phase)
**Requirements**: INFR-01, INFR-02, INFR-03, INFR-04, INFR-05
**Success Criteria** (what must be TRUE):
  1. `npm run start:dev` in nemea-back/ boots without errors and responds to `GET /health`
  2. `npm run migration:generate` and `npm run migration:run` execute without errors against local Docker PostgreSQL
  3. `synchronize: false` is hardcoded unconditionally in TypeORM config — verified by code review
  4. Swagger UI is accessible at `http://localhost:4000/api/docs` in development
  5. Railway deploy succeeds and `GET /health` returns 200 in the production environment
**Plans**: 3 plans

Plans:
- [x] 01-01-PLAN.md -- NestJS 11 project scaffold, TypeScript strict, ESLint flat config, Prettier, Husky
- [x] 01-02-PLAN.md -- TypeORM + Docker Compose PostgreSQL + shared DataSource + BaseEntity + migrations
- [x] 01-03-PLAN.md -- Swagger, helmet, throttler, env validation, HttpExceptionFilter, health check, Railway Procfile

### Phase 2: Auth
**Goal**: Users can log in with Google and the backend enforces authentication and roles on every request
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
**Success Criteria** (what must be TRUE):
  1. User can click "Sign in with Google" and arrive at the app in an authenticated session that persists across browser restarts
  2. A user whose email is NOT in the whitelist table is refused access and receives a 401 — even with a valid Google account
  3. An ADMIN user can reach write endpoints; a USER role user receives 403 on those same endpoints
  4. Removing an email from the whitelist in the DB causes the next API request from that user to fail (no waiting for JWT expiry)
  5. All backend routes return 401 when called without a valid Authorization header (global guard, no opt-in required)
**Plans**: 3 plans

Plans:
- [x] 02-01-PLAN.md -- User entity + migration with admin seed, UsersModule/UsersService, auth decorators (@Public/@Roles/@CurrentUser), JwtAuthGuard + RolesGuard as APP_GUARD, JWT strategy with whitelist check, AuthModule wiring
- [x] 02-02-PLAN.md -- AuthService (Google id_token verification + JWT minting), AuthController (POST /auth/google, GET /auth/me), UsersController (whitelist CRUD, ADMIN only), E2E tests
- [x] 02-03-PLAN.md -- Auth.js v5 NextAuth config (Google provider, jwt/session callbacks, token exchange), proxy.ts route protection, login page with Nemea branding, access-denied page, SessionProvider, api.ts fetch wrapper

### Phase 3: Catalogs and Suppliers
**Goal**: All reference data (product dimensions, supply types, suppliers) exists and can be managed before any supply or product is created
**Depends on**: Phase 2
**Requirements**: CATL-01, CATL-02, CATL-03, CATL-04, CATL-05, CATL-06, SUPP-01, SUPP-02
**Success Criteria** (what must be TRUE):
  1. Admin can create, edit, and delete product types, names, finishes, colors, and sizes via the UI
  2. Admin can create, edit, and delete supply types via the UI
  3. Admin can create a supplier with name, address, email, phone, whatsapp, and description
  4. Admin can deactivate a supplier and the supplier no longer appears in active lists (soft delete)
  5. USER role user can view all catalog and supplier data but cannot modify anything
**Plans**: 4 plans

Plans:
- [x] 03-01-PLAN.md -- UUID migration (users integer->UUID) + BaseEntity update to UUID PK + fix all id:number references across backend and frontend
- [x] 03-02-PLAN.md -- 6 catalog entities + supplier entity + migrations + CatalogsModule (generic CRUD) + SuppliersModule (CRUD + toggle-status) + seed data + AppModule registration
- [x] 03-03-PLAN.md -- Shadcn sidebar + route groups (auth/app) + catalog tabs page with inline CRUD + supplier list/form pages + client API wrapper + visual verification checkpoint
- [x] 03-04-PLAN.md -- Gap closure: fix supplier email validation chain, revert catalog tabs scroll (flex-wrap kept), confirm mobile sidebar backdrop

### Phase 4: Supplies and Price History
**Goal**: Supplies with their types and suppliers exist, and every price change is preserved as an immutable historical record
**Depends on**: Phase 3
**Requirements**: SPPL-01, SPPL-02, SPPL-03, SPPL-04, SPPL-05
**Success Criteria** (what must be TRUE):
  1. Admin can create a supply by selecting its type and supplier
  2. Admin can add a new price to a supply; the previous price record is preserved and still visible in history
  3. User can view all historical prices for any supply sorted from newest to oldest
  4. The "current price" of a supply is always the most recently added record — verified by checking the product cost after a price update
  5. Admin can deactivate a supply and it is excluded from cost calculations going forward
**Plans**: 3 plans (2 feature + 1 gap closure)

Plans:
- [x] 04-01-PLAN.md -- Supply + SupplyPriceHistory entities, migration (partial unique index + composite price index), SuppliesModule CRUD + price endpoints, supplier cascade deactivation
- [x] 04-02-PLAN.md -- Frontend grouped expandable table by supply type, create/edit modal, inline price addition, price history modal, search/filter, sidebar "Datos base" group
- [x] 04-03-PLAN.md -- Gap closure: fix toggleStatus supplier validation, Next.js dev server error, edit modal stale data on reopen

### Phase 5: Products and BOM
**Goal**: Products exist with auto-generated SKUs, their material composition is defined and versioned, and their selling price is tracked
**Depends on**: Phase 4
**Requirements**: PROD-01, PROD-02, PROD-03, PROD-04, PROD-05, PROD-06, PROD-07
**Success Criteria** (what must be TRUE):
  1. Admin can create a product by selecting type, name, finish, color, and size — a unique SKU is generated automatically
  2. Admin can define the BOM (list of supplies with quantity and unit_type) for a product
  3. When Admin changes the BOM, the old composition is preserved with is_active = false and the new one becomes active — old record is still queryable
  4. Admin can add a selling price to a product; previous selling prices are preserved as history
  5. Admin can deactivate a product; it is excluded from active product views but the record is not deleted
**Plans**: 4 plans

Plans:
- [x] 05-01-PLAN.md -- sku_code on catalog tables + Product entity + migration + ProductsModule CRUD + SKU generation + batch creation
- [x] 05-02-PLAN.md -- BOM entity + ProductPriceHistory entity + migrations + BOM endpoints + price endpoints + supply deactivation guard + seed data
- [x] 05-03-PLAN.md -- Frontend product list grouped by type, batch creation modal, edit modal, expand inline with BOM/price, sidebar update
- [x] 05-04-PLAN.md -- BOM editor modal, group BOM editor, selling price inline/history/batch, visual verification checkpoint

### Phase 6: Cost Calculation
**Goal**: Every product displays its current real cost calculated dynamically from the latest supply prices — the core value of the app
**Depends on**: Phase 5
**Requirements**: COST-01, COST-02, COST-03, COST-04
**Success Criteria** (what must be TRUE):
  1. The product list shows a calculated cost column for every product, computed from current supply prices
  2. After updating a supply price, the product list immediately reflects the new cost for all products that use that supply — no manual action required
  3. The product detail page shows a cost breakdown listing each supply, its quantity, its current price, and its line cost
  4. NestJS logs confirm cost calculation uses exactly 2 SQL queries for the entire product list (no N+1 queries)
**Plans**: 2 plans

Plans:
- [x] 06-01-PLAN.md -- CostsModule + CostsService with batched DISTINCT ON query + enrich GET /products and GET /products/:id with cost data
- [x] 06-02-PLAN.md -- Frontend: cost/margin columns in product list, enriched BOM table, product detail page at /productos/:id, group header aggregation

### Phase 7: Expenses
**Goal**: Expense tracking is operational at parity with the Google Sheets
**Depends on**: Phase 6
**Requirements**: EXPN-01, EXPN-02, EXPN-03
**Success Criteria** (what must be TRUE):
  1. Admin can register an expense with amount, concept, date, and category (selected from editable categories)
  2. Admin can view the expense list grouped by month, filtered by category or date range, with subtotals per month and a grand total summary
**Plans**: 2 plans

Plans:
- [x] 07-01-PLAN.md -- ExpenseCategory as catalog dimension + Expense entity + ExpensesModule CRUD with filtering + migration with seed data
- [x] 07-02-PLAN.md -- Frontend: expense list grouped by month, create/edit/delete modals, category/date filters, summary bar, catalog tab, sidebar restructure + visual verification

</details>

### Phase 8: Hardening
**Goal**: The app is secure, DRY, and investment-ready — all existing endpoints enforce roles, JWT expiry is handled gracefully, and shared types eliminate duplication
**Depends on**: Phase 7
**Requirements**: HARD-01, HARD-02, HARD-03, HARD-04, HARD-05, HARD-06, HARD-07
**Success Criteria** (what must be TRUE):
  1. Unauthenticated user navigating to any /app route is redirected to /login by Next.js middleware — no flash of protected content
  2. When a JWT expires mid-session, the user is redirected to /login automatically instead of seeing a broken UI or generic error
  3. A USER-role investor calling a mutation endpoint (POST, PATCH, DELETE) on products, supplies, suppliers, catalogs, or expenses receives a 403 Forbidden
  4. Admin can view a /usuarios page listing all users, create a new user with email and role, and deactivate an existing user
  5. Shared types (SupplyOption, UNIT_LABELS, formatDate, cleanSupplierData) exist in exactly one location each — grep confirms zero duplicates
**Plans**: 2 plans

Plans:
- [x] 08-01-PLAN.md -- Auth middleware fix (proxy.ts -> middleware.ts), 401/403 handling in apiClientFetch, role-based routing, user toggle-status endpoint, dead code cleanup, produccion externa seed
- [x] 08-02-PLAN.md -- DRY extraction (SupplyOption, UNIT_LABELS, formatDate, cleanSupplierData, SupplyCombobox), users admin page, sidebar admin section, expense form fix

### Phase 9: Product UX
**Goal**: Products are navigable by business hierarchy (type > name > finish) and BOM editing respects the real grouping level (name, not type)
**Depends on**: Phase 8 (requires HARD-07 shared types)
**Requirements**: PRUX-01, PRUX-02, PRUX-03
**Success Criteria** (what must be TRUE):
  1. Product table displays a collapsible tree: type level collapses to show names, name level collapses to show finishes, with product rows nested under finish
  2. BOM group editor triggered from a product name header applies to all products sharing that name (e.g., all "Hercules" regardless of type), not all products of the same type
  3. An individual product can have a custom BOM that differs from its name-group default, and the UI clearly indicates which products have overrides
**Plans**: 1 plan

Plans:
- [x] 09-01-PLAN.md -- Hierarchical product tree (Type > Name > Finish), BOM group editor rescoped to name level, BOM override indicator badge

### Phase 10: Tiendanube Config
**Goal**: Tiendanube rate tables are stored in the database and editable by admin, serving as the single source of truth for all pricing calculations
**Depends on**: Phase 8
**Requirements**: TNCF-01, TNCF-02, TNCF-03, TNCF-04
**Success Criteria** (what must be TRUE):
  1. Admin can view and edit the four Tiendanube plans (Inicial, Esencial, Impulso, Escala) with their commission rates per payment method and deposit timing
  2. Admin can view and edit installment fee rates for 1, 3, 6, 9, and 12 cuotas
  3. Admin can view and edit tax configuration (IVA rate, IIBB alicuota, transaction fee per plan)
  4. The config page shows a "Verificar tasas" link that opens the official Pago Nube rates page in a new tab
  5. Seed migration populates the tables with verified March 2026 rates so the system is usable immediately after deploy
**Plans**: 3 plans

Plans:
- [x] 10-01-PLAN.md -- TiendanubeConfigModule: 5 entities (gateways, rates, installments, tax config, plans), migration with March 2026 seed data, CRUD service with append-only history, REST controller, AppModule registration
- [x] 10-02-PLAN.md -- Frontend: /configuracion/tiendanube page with plan selector dropdown, collapsible gateway sections, plan CPT editor, installment editor, tax editor, "Verificar tasas" link, sidebar Admin link
- [ ] 10-03-PLAN.md -- Gap closure: fix raw SQL snake_case column aliases in getLatestGatewayRates + getInstallmentRates, remove calculadora snake_case workarounds, update test mocks

### Phase 11: Calculadora
**Goal**: Users can simulate Tiendanube pricing for any product — seeing real profit after all deductions (forward) or the price needed for a target profit (inverse)
**Depends on**: Phase 10
**Requirements**: CALC-01, CALC-02, CALC-03, CALC-04
**Success Criteria** (what must be TRUE):
  1. User selects a product and a Tiendanube plan, and sees net profit calculated from real cost (from DB) minus all Tiendanube deductions (commission, installments, IVA, IIBB)
  2. User enters a desired profit amount and the calculadora returns the required selling price — the forward calculation of that price matches the target profit within $0.01
  3. Product cost is auto-populated from the database (not manually entered) and updates if the underlying supply prices change
  4. A batch endpoint returns margins for all products in the catalog in a single request, using no more than 6 SQL queries regardless of product count
**Plans**: 3 plans

Plans:
- [x] 11-01-PLAN.md -- CalculadoraModule: service with calcForward/calcInverse/calcBatch (TDD with Hefesto $87,000 round-trip), DTOs, controller with 3 POST endpoints, AppModule registration
- [x] 11-02-PLAN.md -- Frontend: /calculadora page with two-column layout, product selector with cost auto-fill, cascading gateway selectors, mode toggle (forward/inverse), full desglose panel, sidebar link
- [ ] 11-03-PLAN.md -- Gap closure: fix infinite re-render loop in GatewaySelectors + CalculadoraClient (useCallback + ref-based callback pattern)

### Phase 12: Scenarios
**Goal**: Investors can create what-if scenarios with overridden selling prices and see recalculated margins without touching real data
**Depends on**: Phase 11
**Requirements**: SCEN-01, SCEN-02, SCEN-03, SCEN-04
**Success Criteria** (what must be TRUE):
  1. User can create a named scenario (e.g., "Aumento Enero 2027") and set custom selling prices for specific products within it
  2. User can apply a bulk adjustment (e.g., "+10% all cinturones") and the scenario recalculates all affected product margins instantly
  3. Scenario shows a margin summary table with real costs, overridden prices, and recalculated net margins after Tiendanube deductions
  4. Each user sees only their own scenarios — querying another user's scenario returns 404
  5. Creating or modifying a scenario never writes to product_price_history — real pricing data remains untouched
**Plans**: 3 plans

Plans:
- [x] 12-01-PLAN.md -- ScenariosModule: 2 entities (scenario, scenario_override), migration, CRUD service with user-scoped visibility, calculate endpoint delegating to calcForward, REST controller with 8 endpoints, AppModule registration
- [x] 12-02-PLAN.md -- Frontend: /escenarios list page with create/delete/toggle-public, /escenarios/[id] editor with product override table, bulk override dialog, gateway/plan selectors, margin comparison, sidebar link
- [x] 12-03-PLAN.md -- Gap closure: fix inactive products not shown in editor, admin delete for public scenarios from other users

### Phase 12.3: Scenarios v2 — cost overrides and product grouping (INSERTED)

**Goal:** [Urgent work - to be planned]
**Requirements**: TBD
**Depends on:** Phase 12
**Plans:** 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 12.3 to break down)

### Phase 12.2: UI polish and product page UX (INSERTED)

**Goal:** [Urgent work - to be planned]
**Requirements**: TBD
**Depends on:** Phase 12
**Plans:** 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 12.2 to break down)

### Phase 12.1: Dynamic roles with configurable permissions (INSERTED)

**Goal:** Replace hardcoded ADMIN/USER enum with dynamic roles table featuring 11 boolean permission flags, enabling admin to create custom roles and assign granular permissions via a dedicated management page
**Requirements**: D-01, D-02, D-03, D-04, D-05, D-06, D-07, D-08, D-09, D-10, D-11, D-12, D-13, D-14, D-15
**Depends on:** Phase 12
**Success Criteria** (what must be TRUE):
  1. Role entity exists with 11 boolean permission flags, seeded ADMIN (all true) and USER (view + calc + scenarios) roles
  2. PermissionsGuard replaces RolesGuard globally, checking permission flags via @RequirePermission decorator
  3. All 40 controller endpoints use @RequirePermission instead of @Roles
  4. Frontend usePermissions() hook returns all 11 flags from session, middleware enforces per-route permissions
  5. Admin can create, edit, and delete custom roles via /roles page with permission toggles
  6. Users page uses dynamic role selection from API instead of hardcoded admin/user dropdown
**Plans:** 4 plans

Plans:
- [x] 12.1-01-PLAN.md — Backend foundation: Role entity + 11 boolean flags, RolesModule CRUD, breaking migration, PermissionsGuard + @RequirePermission decorator, JWT with embedded permissions, User entity ManyToOne Role
- [ ] 12.1-02-PLAN.md — Backend controller migration: Replace all 40 @Roles() with @RequirePermission(), fix scenarios admin bypass, delete old auth files
- [ ] 12.1-03-PLAN.md — Frontend permission system: Permissions type, usePermissions() hook, NextAuth session extension, middleware ROUTE_PERMISSIONS map, replace isAdmin in all 16 files + sidebar
- [ ] 12.1-04-PLAN.md — Roles management UI: /roles page with table + create/edit/delete dialogs, /usuarios page with dynamic role selection, visual verification

### Phase 13: Investor Dashboard
**Goal**: Investors have a single page showing the full catalog with margins, aggregate metrics, and scenario-aware views
**Depends on**: Phase 12
**Requirements**: DASH-01, DASH-02, DASH-03
**Success Criteria** (what must be TRUE):
  1. Investor sees a catalog summary table showing every active product with its cost, current selling price, and net margin after Tiendanube deductions
  2. Dashboard can be filtered by product type, and the filtered view updates the aggregate metrics accordingly
  3. Dashboard shows aggregate KPIs: average margin percentage, total catalog value, and the products with best and worst margins
  4. Dashboard loads with no more than 6 SQL queries total regardless of product count — no N+1 performance degradation
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10 -> 11 -> 12 -> 12.1 -> 12.2 -> 12.3 -> 13

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Backend Scaffold | v1.0 | 3/3 | Complete | 2026-02-28 |
| 2. Auth | v1.0 | 3/3 | Complete | 2026-03-01 |
| 3. Catalogs and Suppliers | v1.0 | 4/4 | Complete | 2026-03-01 |
| 4. Supplies and Price History | v1.0 | 3/3 | Complete | 2026-03-05 |
| 5. Products and BOM | v1.0 | 4/4 | Complete | 2026-03-06 |
| 6. Cost Calculation | v1.0 | 2/2 | Complete | 2026-03-11 |
| 7. Expenses | v1.0 | 2/2 | Complete | 2026-03-11 |
| 8. Hardening | v1.1 | 2/2 | Complete | 2026-03-27 |
| 9. Product UX | v1.1 | 1/1 | Complete | 2026-03-27 |
| 10. Tiendanube Config | v1.1 | 2/3 | UAT gap closure | - |
| 11. Calculadora | v1.1 | 2/3 | UAT gap closure | - |
| 12. Scenarios | v1.1 | 3/3 | Complete | 2026-03-29 |
| 12.1. Dynamic Roles | v1.1 | 1/4 | In progress | - |
| 13. Investor Dashboard | v1.1 | 0/? | Not started | - |
