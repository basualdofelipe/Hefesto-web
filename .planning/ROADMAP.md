# Roadmap: Nemea

## Overview

Nemea is a cost management system for an artisan leather goods workshop that replaces Google Sheets.
The journey: stand up a production-grade NestJS backend, lock down auth, build catalogs and supplier
data as the FK foundation, layer supplies with price history, define product BOMs, wire up dynamic cost
calculation (the core value), and finish with expenses and Tiendanube config. The frontend is already
scaffolded — each phase delivers working backend endpoints consumed by real UI.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Backend Scaffold** - NestJS 11 app running with TypeORM migrations, Docker Compose, Swagger
- [x] **Phase 2: Auth** - Google OAuth via NextAuth exchanges for NestJS JWT, global guard, email whitelist, roles
- [ ] **Phase 3: Catalogs and Suppliers** - CRUD for all 5 product dimensions, supply types, and suppliers
- [ ] **Phase 4: Supplies and Price History** - Supply CRUD with append-only price history and composite index
- [ ] **Phase 5: Products and BOM** - Product CRUD with SKU, material composition with version history, selling price
- [ ] **Phase 6: Cost Calculation** - Dynamic cost per product (batched DISTINCT ON query), visible in product list and detail
- [ ] **Phase 7: Expenses and Config** - Expense tracking with categories, Tiendanube config storage

## Phase Details

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
**Plans**: 3 plans

Plans:
- [x] 03-01-PLAN.md -- UUID migration (users integer→UUID) + BaseEntity update to UUID PK + fix all id:number references across backend and frontend
- [x] 03-02-PLAN.md -- 6 catalog entities + supplier entity + migrations + CatalogsModule (generic CRUD) + SuppliersModule (CRUD + toggle-status) + seed data + AppModule registration
- [ ] 03-03-PLAN.md -- Shadcn sidebar + route groups (auth/app) + catalog tabs page with inline CRUD + supplier list/form pages + client API wrapper + visual verification checkpoint

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
**Plans**: TBD

Plans:
- [ ] 04-01: Supplies entity + migration (with soft delete partial index) + CRUD endpoints + SuppliesService exported
- [ ] 04-02: SuppliesPriceHistory entity + migration (composite index on supply_id, created_at DESC) + POST /supplies/:id/prices + GET /supplies/:id/prices
- [ ] 04-03: Frontend pages: supplies list, supply detail with price history, add price form

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
**Plans**: TBD

Plans:
- [ ] 05-01: Catalog dimension entities wired to products, Product entity + migration (5 FK dims + sku_code + is_active) + SKU generation + CRUD endpoints
- [ ] 05-02: SuppliesPerProductHistory entity + migration (BOM with is_active versioning, transaction wrapping) + BOM endpoints
- [ ] 05-03: ProductPriceHistory entity + migration + POST /products/:id/prices endpoint + GET /products/:id/prices
- [ ] 05-04: Frontend pages: product list (no cost yet), product detail, create/edit product form, BOM editor

### Phase 6: Cost Calculation
**Goal**: Every product displays its current real cost calculated dynamically from the latest supply prices — the core value of the app
**Depends on**: Phase 5
**Requirements**: COST-01, COST-02, COST-03, COST-04
**Success Criteria** (what must be TRUE):
  1. The product list shows a calculated cost column for every product, computed from current supply prices
  2. After updating a supply price, the product list immediately reflects the new cost for all products that use that supply — no manual action required
  3. The product detail page shows a cost breakdown listing each supply, its quantity, its current price, and its line cost
  4. NestJS logs confirm cost calculation uses exactly 2 SQL queries for the entire product list (no N+1 queries)
**Plans**: TBD

Plans:
- [ ] 06-01: CostsModule + CostsService with batched DISTINCT ON query (1 composition query + 1 price query + in-memory calc)
- [ ] 06-02: GET /products returns ProductWithCostDto[] with cost and cost_breakdown, role-differentiated view for USER
- [ ] 06-03: Frontend: product list with cost column, product detail with cost breakdown table

### Phase 7: Expenses and Config
**Goal**: Expense tracking is operational at parity with the Google Sheets, and Tiendanube configuration is stored in the DB for future use
**Depends on**: Phase 6
**Requirements**: EXPN-01, EXPN-02, EXPN-03, CONF-01, CONF-02
**Success Criteria** (what must be TRUE):
  1. Admin can register an expense with amount, concept, date, and category (one of 6 predefined categories)
  2. Admin can view the expense list filtered by category or by date range
  3. Admin can view and edit the Tiendanube configuration (plan, card rates, transfer rate, installments, IIBB, IVA) and the values persist in the DB
  4. USER role user can view expenses and config but cannot create or modify either
**Plans**: TBD

Plans:
- [ ] 07-01: Expenses entity + migration (amount, concept, date, category enum) + CRUD endpoints with filters
- [ ] 07-02: TiendanubeConfigModule entity + migration + GET/PUT /config/tiendanube (named TiendanubeConfigModule to avoid @nestjs/config collision)
- [ ] 07-03: Frontend pages: expense list with filters, add expense form, Tiendanube config settings page

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Backend Scaffold | 3/3 | Complete (pending verification) | 2026-02-28 |
| 2. Auth | 3/3 | Complete | 2026-03-01 |
| 3. Catalogs and Suppliers | 2/3 | In progress | - |
| 4. Supplies and Price History | 0/3 | Not started | - |
| 5. Products and BOM | 0/4 | Not started | - |
| 6. Cost Calculation | 0/3 | Not started | - |
| 7. Expenses and Config | 0/3 | Not started | - |
