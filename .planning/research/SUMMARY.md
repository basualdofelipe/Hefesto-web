# Project Research Summary

**Project:** Nemea — cost management system for artisan leather goods workshop
**Domain:** Product cost management / inventory with price history (SMB artisan)
**Researched:** 2026-02-28
**Confidence:** MEDIUM-HIGH

## Executive Summary

Nemea is a bespoke internal tool that replaces a Google Sheets + Apps Script system for an artisan leather goods business. The domain is product cost management: tracking raw material prices over time, defining bill-of-materials (BOM) per product, and computing current cost automatically when prices change. Research confirms the already-chosen stack (NestJS 11 + TypeORM + PostgreSQL + Next.js + NextAuth) is production-grade and correct. No technology changes are recommended. The biggest open question was the auth integration between NextAuth and NestJS — that pattern is now fully documented: NextAuth handles Google OAuth, exchanges the Google `id_token` for a NestJS-minted JWT at login, and sends that NestJS JWT as `Authorization: Bearer` on every API call. The backend never calls Google on each request.

The architectural approach is a clean modular NestJS application with 5 domain modules (Auth, Supplies, Products, Costs, TiendanubeConfig) backed by PostgreSQL with append-only price history and an `is_active` BOM versioning pattern. Cost calculation is performed on-the-fly at query time using a single batched SQL query — no cache needed at this scale (2-3 users, ~100 products). The CostsModule is deliberately separated from ProductsModule because cost calculation is a cross-cutting read concern that depends on both supplies and products.

The top risks are TypeORM migration misuse (`synchronize: true` causing silent data loss in production), the auth token exchange complexity between NextAuth and NestJS, and the N+1 query pattern when computing latest price per supply. All three are entirely preventable with the patterns documented in STACK.md and ARCHITECTURE.md, and all three must be addressed in their respective setup phases rather than retrofitted later.

---

## Key Findings

### Recommended Stack

The stack is already decided and validated. NestJS 11 is the current stable release (Jan 2025, supported until 2030+). TypeORM 0.3.28 is the active API — the 0.3.x DataSource pattern is what the CLI expects. PostgreSQL 16-alpine on Docker for local development, Railway for production. Node 20 LTS matches the frontend. All `@nestjs/*` packages must match major version 11.

One non-obvious gap in the original stack definition: `passport-google-oauth20` is NOT needed on the backend. The backend only validates JWTs — it never runs an OAuth flow. The only auth libraries needed are `@nestjs/passport`, `passport`, `passport-jwt`, and `@nestjs/jwt`. A shared `NEXTAUTH_SECRET` between the Next.js and NestJS apps is the key integration point.

**Core technologies:**
- NestJS 11: Backend framework — current stable, 2030+ support, improved module startup performance
- TypeORM 0.3.28: ORM for PostgreSQL — active API (DataSource), matches Freedom-Base reference pattern
- PostgreSQL 16-alpine: Primary database — current stable, Docker locally / Railway in production
- `@nestjs/passport` + `passport-jwt`: JWT guard — validates NextAuth-signed tokens without calling Google per request
- `@nestjs/config` + Joi: Environment management — validates required env vars at startup, fails fast on misconfiguration
- `class-validator` + `class-transformer`: DTO validation — standalone packages only (not the unmaintained `@nestjs/` forks)
- `helmet` + `@nestjs/throttler`: Security baseline — HTTP headers and rate limiting, applied globally at bootstrap
- `@nestjs/swagger`: API documentation — auto-generated from decorators, enable in dev only

See `.planning/research/STACK.md` for full library versions, Docker Compose config, migration setup scripts, and Railway production configuration.

### Expected Features

Nemea's feature set is a lean intersection of what artisan cost management tools (Craftybase, Katana MRP) provide, stripped to what actually maps to this business's current operations. The risk is building MORE than needed, not less — the data model and business rules are already fully known from the existing Sheets.

**Must have (table stakes — Sheets replacement is incomplete without these):**
- Google OAuth authentication with email whitelist and ADMIN/USER roles — prerequisite for everything
- Catalog CRUD (5 product dimensions: type, name, finish, color, size — plus supply types and suppliers) — prerequisite for all entity CRUDs
- Supply CRUD with append-only price history — foundation of cost calculations
- Product CRUD with 5-dimension model and auto-generated SKU code
- BOM management (material composition per product) with version history via `is_active` flag
- Dynamic cost calculation surfaced in the products list view — the core value proposition
- Expense tracking with categories — parity with Sheets
- Selling price history per product — needed to compute margin
- Investor read-only view (same product list, restricted role) — named stakeholder requirement
- Soft delete for products and supplies — data integrity from day one

**Should have (differentiators, add in v1.x after core is stable):**
- Cost breakdown by material type (leather vs hardware vs packaging subtotals in product detail)
- "Stale price" indicator on supplies (price not updated in N days)
- CSV export of product list with costs

**Defer (v2+):**
- Tiendanube config + calculadora — user explicitly descoped from v1
- B2B clients and orders — v2
- Investor dashboard with computed metrics — v2
- Multi-currency (ARS/USD) prices — v2 if operationally needed
- Stock quantity tracking — v3+ (requires orders/fulfillment data model first)
- Labor cost in BOM — v3+ (requires time-tracking process that does not exist)
- Data migration from Google Sheets — final v1 phase, after app is stable

**Key dependency chain (build order constraint):**
Auth → Catalog → Suppliers → Supplies + Price History → Products + BOM → Cost Calculation

See `.planning/research/FEATURES.md` for full prioritization matrix and anti-feature analysis.

### Architecture Approach

The system follows a standard NestJS layered architecture (controller → service → repository) with 5 business modules. The most important architectural decision is that CostsModule is separate from ProductsModule: it imports both SuppliesModule and ProductsModule to perform enrichment, so ProductsModule never imports CostsModule (avoiding circular dependencies). Auth is implemented as a global `APP_GUARD` — every route is protected by default, with an explicit `@Public()` decorator as an opt-out escape hatch.

**Major components:**
1. **AuthModule** — JWT validation (NextAuth-signed tokens), email whitelist check on every request, role attachment to request context. Two guards registered globally: `JwtAuthGuard` and `RolesGuard`.
2. **SuppliesModule** — CRUD for supplies, suppliers, supply types, and append-only price history. Exports `SuppliesService` for use by CostsModule.
3. **ProductsModule** — CRUD for products, 5 catalog dimension tables, BOM management with `is_active` versioning, selling price history. Exports `ProductsService` for use by CostsModule.
4. **CostsModule** — Cross-cutting read service. Imports SuppliesModule + ProductsModule. Implements batched cost calculation: 1 query for active compositions, 1 `DISTINCT ON` query for latest prices per supply, in-memory multiplication. No N+1 queries.
5. **TiendanubeConfigModule** — CRUD for Tiendanube rate configuration (explicitly named to avoid collision with `@nestjs/config`'s `ConfigModule`).

See `.planning/research/ARCHITECTURE.md` for full directory structure, all 4 data flows with TypeScript examples, and scaling considerations.

### Critical Pitfalls

1. **`synchronize: true` in TypeORM reaching production** — hardcode `synchronize: false` unconditionally (never `NODE_ENV !== 'production'`). Use `migrationsRun: true` as the safe equivalent. Set this up in Phase 2 before writing a single entity. Failure mode: silent column drops or table destruction on deploy.

2. **Two TypeORM DataSource configs falling out of sync** — do NOT use `autoLoadEntities: true` in `TypeOrmModule.forRoot()`. Import the same entities array from `src/database/dataSource.ts` in both NestJS config and CLI config. Failure mode: `migration:generate` produces empty files, entity silently missing from DB.

3. **NextAuth token exchange done incorrectly** — NextAuth `jwt` callback must call `POST /auth/google` on first login with Google's `id_token`, receive a NestJS-minted JWT, and store it as `token.backendToken`. Frontend sends this NestJS JWT as `Authorization: Bearer`. NestJS never calls Google per request. Failure mode: 401s on all API calls, or 200-400ms latency per request if Google verification runs on each.

4. **Email whitelist checked only at login** — `JwtAuthGuard` must query the database whitelist on every request (one indexed lookup). Not just at login. Failure mode: deactivated users retain API access for the full JWT lifetime.

5. **Soft delete breaking unique constraints** — `is_active = false` records still satisfy standard `UNIQUE` constraints, preventing re-creation of same-name supplies. Prevention: create partial unique indexes in the same migration that creates the table (`WHERE is_active = true`), or use TypeORM's `@DeleteDateColumn` with `WHERE deleted_at IS NULL`. Must be decided in Phase 4 before any data exists.

See `.planning/research/PITFALLS.md` for full pitfall-to-phase mapping, security mistakes checklist, and recovery strategies.

---

## Implications for Roadmap

Based on combined research, the phase structure is directly driven by the feature dependency chain and module build order from ARCHITECTURE.md. No phase can be freely reordered.

### Phase 1: Backend Scaffold

**Rationale:** The backend does not exist yet (Fase 2 in MEMORY.md). All subsequent phases depend on a correctly configured NestJS application with migrations wired up before any entity is written.
**Delivers:** NestJS 11 app scaffolded at `nemea-back/`, PostgreSQL running via Docker Compose, TypeORM migrations configured (`synchronize: false`, `migrationsRun: true`), Swagger available at `/api/docs`, Railway environment variables documented.
**Addresses features:** Infrastructure only — no business features.
**Avoids pitfalls:** Pitfall 1 (`synchronize: true`) and Pitfall 2 (DataSource config divergence) must be solved here, not retrofitted.
**Research flag:** Standard patterns — no additional research needed. STACK.md has the exact migration setup scripts.

### Phase 2: Authentication

**Rationale:** Auth is the prerequisite for every other endpoint. No business feature can be built without knowing the auth guard pattern. The Google OAuth ↔ NestJS token exchange (Pitfall 3) is the highest-complexity item in the entire system and must be understood and tested in isolation before any business logic is layered on top.
**Delivers:** `POST /auth/google` endpoint (validates Google `id_token`, mints NestJS JWT), `GET /auth/me`, global `JwtAuthGuard` + `RolesGuard`, email whitelist in `users` table (DB-checked on every request), `ADMIN`/`USER` role enum.
**Addresses features:** Google OAuth authentication, email whitelist, role-based access.
**Avoids pitfalls:** Pitfall 3 (token exchange), Pitfall 4 (whitelist on every request).
**Research flag:** Needs careful implementation — the NextAuth jwt/session callback pattern is documented in PITFALLS.md but is integration-heavy and should be verified end-to-end before proceeding.

### Phase 3: Supplies and Price History

**Rationale:** Supplies are the foundation of cost calculation. CostsModule cannot function without supply prices. Suppliers and supply types must exist (FK constraints) before supplies can be created. This phase also establishes the append-only price history pattern and must include the composite index on `supplies_price_history(supply_id, created_at DESC)` to prevent the N+1 query pitfall.
**Delivers:** Full CRUD for suppliers, supply types, and supplies. Append-only `POST /supplies/:id/prices` endpoint. `GET /supplies/:id/prices` returning history sorted DESC. Composite index on price history table.
**Addresses features:** Supply CRUD with type and supplier, price history per supply, supplier CRUD with contact info, soft delete for supplies.
**Avoids pitfalls:** Pitfall 5 (N+1 price query — index added in migration), Pitfall 6 (soft delete + unique constraints — partial index decision).
**Research flag:** Standard NestJS CRUD patterns — no additional research needed.

### Phase 4: Products and Catalog

**Rationale:** Products depend on the 5 catalog dimension tables having data. Catalog must be seeded with the owner's real data (product types, names, finishes, colors, sizes) before the first product can be created. BOM management (material composition) depends on both supplies (from Phase 3) and products.
**Delivers:** CRUD for all 5 catalog dimension tables, product CRUD with auto-generated SKU from dimension IDs, BOM management with `is_active` version history (transaction: deactivate old rows + insert new), selling price history per product.
**Addresses features:** Product CRUD with 5-dimension catalog, SKU auto-generation, BOM management with version history, selling price history.
**Uses:** Composition history pattern (Pattern 4 from ARCHITECTURE.md), transaction wrapping for BOM changes.
**Research flag:** Standard patterns — SKU generation algorithm needs a brief implementation decision (abbreviation vs ID encoding) but no external research needed.

### Phase 5: Cost Calculation and Integration

**Rationale:** CostsModule is the last domain module because it imports both SuppliesModule and ProductsModule. It cannot be built until both parent modules are stable. This phase delivers the core value proposition: a product list page that shows each product's calculated cost from current supply prices.
**Delivers:** `CostsService.enrichWithCosts()` with the batched DISTINCT ON query (1 composition query + 1 price query + in-memory calc = 0 N+1 queries). `GET /products` returning `ProductWithCostDto[]` (cost, cost breakdown by supply type, latest price date). Investor-ready view differentiated by role.
**Addresses features:** Dynamic cost calculation, automatic cost propagation, cost breakdown by material type, investor read-only view.
**Avoids pitfalls:** Pitfall 5 (N+1 queries) — the correct batched pattern is the only acceptable implementation.
**Research flag:** Standard PostgreSQL DISTINCT ON pattern — documented in ARCHITECTURE.md. No additional research needed.

### Phase 6: Expenses and Config

**Rationale:** Expense tracking and Tiendanube config are lower-dependency features that can be built after the core cost management system is functional. These are simpler CRUDs with no cross-module dependencies.
**Delivers:** Expense CRUD with category enum (parity with Sheets). `TiendanubeConfigModule` with `GET/PUT /config/tiendanube` (correctly named to avoid `@nestjs/config` collision).
**Addresses features:** Expense tracking with categories, Tiendanube rate configuration.
**Avoids pitfalls:** Anti-Pattern 3 (ConfigModule naming collision — use `TiendanubeConfigModule`).
**Research flag:** Standard patterns — no research needed.

### Phase 7: Frontend Integration

**Rationale:** Once all backend endpoints are stable, integrate the Next.js frontend with the NestJS API. Connect the existing calculator prototype to real config data. Build CRUD views for supplies, products, and expenses.
**Delivers:** Complete working v1 application with all backend endpoints consumed by the frontend. Investor view accessible with USER role.
**Addresses features:** All UI-facing features from v1 scope.
**Research flag:** Frontend patterns are well-established (Next.js App Router + fetch layer). Specific component decisions should follow the existing nemea-front scaffold conventions.

### Phase 8: Data Migration

**Rationale:** Migration from Google Sheets is the final phase, after the application is stable and the owner has validated the data model with real usage. Migrating too early risks building the data model around Sheets quirks.
**Delivers:** Owner's real catalog data, supply prices, product compositions, and expense records loaded into production.
**Research flag:** No framework research needed — this is a scripted one-time data import specific to the Sheets structure documented in `costos-nemea-appscripts.txt`.

### Phase Ordering Rationale

- **Auth before everything:** Every other endpoint requires the JWT guard. Building business modules without auth first means retrofitting guards on every controller.
- **Supplies before Products:** FK constraint — supplies have suppliers (FK) and supply types (FK). Products have BOM entries that reference supply IDs (FK).
- **Products after Catalog:** The 5 dimension tables need seed data before a product can be created.
- **Costs last among domain modules:** CostsModule imports both SuppliesModule and ProductsModule. Building it last is architecturally required.
- **Expenses/Config late:** No dependencies on other business modules. Can be built any time after scaffold, but prioritizing core cost model first maximizes early validation.
- **Migration last:** Avoids data model coupling to existing Sheets structure.

### Research Flags

Phases needing careful verification during implementation:
- **Phase 2 (Auth):** The NextAuth `jwt`/`session` callback + NestJS `POST /auth/google` integration is the single highest-complexity element. Test the full token exchange flow end-to-end before building on top of it.
- **Phase 5 (Costs):** The DISTINCT ON batched query must be verified against NestJS logger output to confirm no N+1 pattern is present. Add this as an explicit acceptance criterion.

Phases with standard well-documented patterns (can proceed without deeper research):
- **Phase 1 (Scaffold):** STACK.md has exact setup scripts.
- **Phase 3 (Supplies):** Standard NestJS CRUD + append-only insert pattern.
- **Phase 4 (Products):** Standard CRUD + transaction pattern for BOM versioning.
- **Phase 6 (Expenses/Config):** Simplest CRUDs in the system.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | NestJS 11, TypeORM 0.3.28, and `@nestjs/swagger` 11.2.6 versions confirmed via WebSearch. Migration setup and Railway SSL pattern confirmed via official docs. |
| Features | HIGH | Features derived primarily from first-party sources (scope-v1.md, business-rules.md, PROJECT.md). Competitor analysis (Craftybase, Katana MRP) validates scope is correctly lean. |
| Architecture | HIGH | Patterns are standard NestJS. DISTINCT ON query pattern is well-documented PostgreSQL. Module dependency graph is deterministic from FK constraints. |
| Pitfalls | MEDIUM-HIGH | Critical pitfalls (synchronize, N+1, soft delete) confirmed via TypeORM GitHub issues and multiple community sources. Auth token exchange pattern is MEDIUM — based on community discussions, not official NextAuth docs for this specific flow. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **NextAuth `id_token` exchange implementation:** The documented pattern (exchange Google `id_token` for NestJS JWT in NextAuth `jwt` callback) is community-confirmed but the exact NextAuth v4 callback signatures should be verified against the actual installed version in `nemea-front`. Validate during Phase 2 implementation.
- **SKU generation algorithm:** Research does not prescribe the exact encoding (dimension abbreviations vs dimension IDs). This is a 30-minute implementation decision, not a research gap. Decide during Phase 4 planning.
- **Soft delete strategy (`is_active` vs `@DeleteDateColumn`):** PITFALLS.md documents both options. The decision between them must be made in Phase 3/4 before any entity with unique constraints is persisted. Recommendation: use `@DeleteDateColumn` for TypeORM's built-in soft-delete support and cleaner partial index syntax.
- **Pagination on product list:** PITFALLS.md notes that fetching the full product list with cost calculation per product breaks at >50 products. V1 scale may reach this. Consider adding pagination to `GET /products` from the start (cursor-based or offset — either is fine at this scale).

---

## Sources

### Primary (HIGH confidence)
- `.claude/docs/planning/scope-v1.md` — V1 module definitions, table list, endpoint list
- `.claude/docs/planning/business-rules.md` — Cost calculation formulas, expense categories, BOM structure
- `.planning/PROJECT.md` — Scope boundaries, v1 vs v2 decisions
- NestJS official docs (guards, modules, validation) — Architecture patterns
- TypeORM 0.3.x official docs (DataSource, migrations) — Migration setup
- PostgreSQL docs (DISTINCT ON, partial indexes) — Query patterns

### Secondary (MEDIUM confidence)
- WebSearch: NestJS 11 release (Trilon announcement Jan 2025) — Version confirmation
- WebSearch: TypeORM 0.3.28 (npm registry) — Version confirmation
- WebSearch: `@nestjs/swagger` 11.2.6 (npm registry, Feb 2026) — Version confirmation
- GitHub: nextauthjs/next-auth Discussion #8884 — NextAuth + custom backend token exchange pattern
- GitHub: typeorm/typeorm Issue #7549 — Soft delete + unique constraint problem
- Medium: "NestJS & TypeORM Migrations in 2025" — Two DataSource configs pattern
- wanago.io: "Soft deletes with PostgreSQL and TypeORM" — `@DeleteDateColumn` + partial index

### Tertiary (MEDIUM-LOW confidence)
- Craftybase, Katana MRP feature sets — Competitor analysis for scope validation
- Railway Help Station community posts — SSL connection configuration
- Docker Compose PostgreSQL readiness articles — `depends_on` + health check pattern

---
*Research completed: 2026-02-28*
*Ready for roadmap: yes*
