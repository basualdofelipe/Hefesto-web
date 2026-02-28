# Architecture Research

**Domain:** Product cost management / inventory with price history (marroquinería)
**Researched:** 2026-02-28
**Confidence:** HIGH (stack is decided, patterns are standard NestJS/PostgreSQL, confirmed via official docs and codebase analysis)

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser Client                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  /productos  │  │  /insumos    │  │  /configuracion      │   │
│  │  (table +    │  │  (ABM +      │  │  (tasas Tiendanube)  │   │
│  │   costos)    │  │   precios)   │  │                      │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘   │
└─────────┼─────────────────┼────────────────────┼────────────────┘
          │ HTTP + JWT       │                    │
┌─────────┼─────────────────┼────────────────────┼────────────────┐
│         │         nemea-front (Vercel)          │                │
│  ┌──────▼──────────────────▼────────────────────▼─────────┐     │
│  │              NextAuth + fetch layer                      │     │
│  │   Google OAuth → JWT token → Authorization header       │     │
│  └──────────────────────────┬───────────────────────────────┘    │
└─────────────────────────────┼──────────────────────────────────── ┘
                              │ REST API (HTTPS)
┌─────────────────────────────┼────────────────────────────────────┐
│                    nemea-back (Railway)          │                │
│  ┌──────────────────────────▼───────────────────────────────┐    │
│  │                      AuthGuard (global)                   │    │
│  │       JWT validation → whitelist check → role attach      │    │
│  └──┬──────────────┬─────────────┬──────────────────────┬───┘    │
│     │              │             │                      │         │
│  ┌──▼───┐  ┌───────▼──┐  ┌──────▼────┐  ┌─────────────▼───┐    │
│  │Supplies│ │Products  │  │  Costs    │  │  Config         │    │
│  │Module  │ │Module    │  │  Module   │  │  Module         │    │
│  └──┬─────┘ └──────┬───┘  └──────┬────┘  └─────────────┬───┘    │
│     │              │             │                      │         │
│  ┌──▼──────────────▼─────────────▼──────────────────────▼───┐    │
│  │                    TypeORM Repositories                    │    │
│  └───────────────────────────┬───────────────────────────────┘    │
└──────────────────────────────┼─────────────────────────────────── ┘
                               │ SQL
┌──────────────────────────────▼────────────────────────────────────┐
│                     PostgreSQL (Railway)                            │
│  supplies  supplies_price_history  product  supplies_per_product_  │
│  suppliers supply_type             product_price_history           │
│  users     product_type/name/...   config_tiendanube               │
└────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Communicates With |
|-----------|----------------|-------------------|
| NextAuth (front) | Google OAuth flow, stores JWT in HTTP-only cookie | Google OAuth, nemea-back /auth/validate |
| AuthGuard (back) | Validates JWT on every request, attaches user+role to request context | UsersRepository (whitelist check) |
| RolesGuard (back) | Checks user.role against endpoint @Roles() decorator | AuthGuard output (request.user) |
| AuthModule | Auth endpoints (/auth/me, /auth/validate), guard wiring | UsersRepository |
| SuppliesModule | CRUD supplies + suppliers + supply_types + price history | TypeORM: supplies, suppliers, supply_type, supplies_price_history |
| ProductsModule | CRUD products + catalog tables + material composition | TypeORM: product, product_*, supplies_per_product_history |
| CostsModule | Calculates cost per product using latest prices and active composition | SuppliesModule (price queries), ProductsModule (composition queries) |
| ConfigModule | CRUD config_tiendanube rates | TypeORM: config_tiendanube |

## Recommended Project Structure

```
nemea-back/src/
├── auth/                        # AuthModule — guards, JWT validation, whitelist
│   ├── auth.module.ts
│   ├── auth.controller.ts       # GET /auth/me, POST /auth/validate
│   ├── auth.service.ts
│   ├── guards/
│   │   ├── jwt-auth.guard.ts    # Validates JWT, attaches user to request
│   │   └── roles.guard.ts       # Checks role against @Roles() decorator
│   ├── decorators/
│   │   ├── roles.decorator.ts   # @Roles('ADMIN')
│   │   └── public.decorator.ts  # @Public() — skips auth guard
│   └── strategies/
│       └── jwt.strategy.ts      # Passport JWT strategy
│
├── supplies/                    # SuppliesModule — insumos + precios
│   ├── supplies.module.ts
│   ├── supplies.controller.ts   # GET/POST /supplies, GET/POST /supplies/:id/prices
│   ├── supplies.service.ts
│   ├── entities/
│   │   ├── supply.entity.ts
│   │   ├── supplier.entity.ts
│   │   ├── supply-type.entity.ts
│   │   └── supply-price-history.entity.ts
│   └── dto/
│       ├── create-supply.dto.ts
│       ├── update-supply.dto.ts
│       └── create-price.dto.ts
│
├── products/                    # ProductsModule — productos + catálogos + composición
│   ├── products.module.ts
│   ├── products.controller.ts   # GET/POST/PUT/DELETE /products
│   ├── products.service.ts
│   ├── catalog.controller.ts    # GET /catalog/product-types, etc.
│   ├── catalog.service.ts
│   ├── entities/
│   │   ├── product.entity.ts
│   │   ├── product-type.entity.ts
│   │   ├── product-name.entity.ts
│   │   ├── product-finish.entity.ts
│   │   ├── product-color.entity.ts
│   │   ├── product-size.entity.ts
│   │   ├── supply-per-product-history.entity.ts
│   │   └── product-price-history.entity.ts
│   └── dto/
│       ├── create-product.dto.ts
│       ├── update-product.dto.ts
│       └── set-materials.dto.ts
│
├── costs/                       # CostsModule — cálculo de costo
│   ├── costs.module.ts
│   ├── costs.service.ts         # calculateProductCost(), calculateAllCosts()
│   └── dto/
│       └── product-with-cost.dto.ts
│
├── config/                      # ConfigModule (negocio) — tasas Tiendanube
│   ├── config.module.ts
│   ├── config.controller.ts     # GET/PUT /config/tiendanube
│   ├── config.service.ts
│   └── entities/
│       └── config-tiendanube.entity.ts
│
├── common/                      # Shared utilities — no business logic here
│   ├── decorators/
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── interceptors/
│   └── types/
│       └── role.enum.ts         # enum Role { ADMIN = 'ADMIN', USER = 'USER' }
│
├── database/                    # TypeORM migrations — never synchronize: true
│   └── migrations/
│       └── 001_initial_schema.ts
│
├── app.module.ts                # Root module — wires all modules + TypeORM + Config
└── main.ts                      # Bootstrap — CORS, helmet, global pipes, port 4000
```

### Structure Rationale

- **auth/**: Isolated from business logic. AuthGuard registered as APP_GUARD (global) in AuthModule so all routes are protected by default. @Public() decorator opts out.
- **costs/**: Separate module, not folded into products/, because cost calculation is a cross-cutting concern that reads from both supplies and products. Keeps ProductsModule clean.
- **config/**: Uses NestJS ConfigModule name collision — rename file/module to `tiendanube-config/` or `app-config/` to avoid clash with @nestjs/config. See pitfall below.
- **common/**: Shared code with zero domain logic. Filters, interceptors, enums only.
- **database/migrations/**: Migrations are the single source of truth for schema. Never use synchronize: true, not even in dev.

## Architectural Patterns

### Pattern 1: Global Auth Guard with @Public() Escape Hatch

**What:** Register AuthGuard as a global APP_GUARD. Every route is protected by default. Endpoints that should be public (e.g., /auth/validate itself) use a @Public() decorator that the guard checks before validating JWT.

**When to use:** Any app where the default is "authenticated" and only a few routes are public. Prevents accidentally forgetting to protect a route.

**Trade-offs:** Simpler than opt-in guards per controller. Requires @Public() on every public route, which is explicit and safe.

**Example:**
```typescript
// auth.module.ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },
  { provide: APP_GUARD, useClass: RolesGuard },
]

// jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) { super(); }

  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context) as boolean;
  }
}

// auth.controller.ts
@Public()
@Post('validate')
async validate(@Body() dto: ValidateDto): Promise<UserResponseDto> { ... }
```

### Pattern 2: Immutable Price History — Append-Only Table

**What:** Never update or delete `supplies_price_history` records. Every price change = new INSERT. The "current price" is always a query for `MAX(date)` or `ORDER BY date DESC LIMIT 1` per supply.

**When to use:** Any domain with audit requirements or where historical cost reconstruction matters. Marroquinería needs this to answer "what did this billetera cost in March?".

**Trade-offs:** Read queries are slightly more complex (need subquery or DISTINCT ON). Write path is simpler (no update logic). PostgreSQL DISTINCT ON is highly efficient with proper index.

**Example — recommended query pattern:**
```typescript
// In CostsService or SuppliesService
// Uses PostgreSQL DISTINCT ON for efficiency — one pass, not N subqueries
async getLatestPricePerSupply(): Promise<LatestPriceRow[]> {
  return this.dataSource.query(`
    SELECT DISTINCT ON (supply_id)
      supply_id,
      price,
      date
    FROM supplies_price_history
    ORDER BY supply_id, date DESC
  `);
}
```

```sql
-- Required index (stated in scope-v1.md, confirmed as best practice)
CREATE INDEX idx_price_history_supply_date
  ON supplies_price_history (supply_id, date DESC);
```

### Pattern 3: On-the-Fly Cost Calculation (no cache for v1)

**What:** When `GET /products` is called, CostsModule calculates each product's cost dynamically: fetch active composition, fetch latest price per supply, multiply and sum.

**When to use:** Small dataset (< 100 products, < 50 supplies), internal app with 2-3 users, no SLA on response time. Correct for Nemea v1.

**Trade-offs:**
- Pro: Always reflects current prices. No cache invalidation complexity. Simpler code.
- Con: O(products × supplies) queries if naively implemented. Mitigated by batching.

**Correct batching approach:**
```
1. Fetch all active products
2. Fetch all active compositions (supplies_per_product_history WHERE is_active = true)
3. Fetch latest price for every supply_id that appears in compositions (single DISTINCT ON query)
4. Calculate costs in memory — no N+1 queries
```

**Anti-pattern to avoid:** Querying latest price per supply inside a loop over products. That's O(N×M) database roundtrips.

### Pattern 4: Composition History with is_active Flag

**What:** `supplies_per_product_history` stores all historical compositions. The "current" composition for a product is all rows WHERE `product_id = ? AND is_active = true`. When composition changes, set old rows to `is_active = false` and INSERT new rows.

**When to use:** When you need to answer "what materials did this product use before the recipe changed?" AND have the current recipe in a simple query.

**Trade-offs:** Slightly more complex write path (deactivate old + insert new in a transaction). Read path is clean (always WHERE is_active = true).

**Example:**
```typescript
// products.service.ts
async setMaterials(productId: number, materials: SetMaterialsDto[]): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    // Deactivate current composition
    await manager.update(
      SupplyPerProductHistory,
      { product_id: productId, is_active: true },
      { is_active: false }
    );
    // Insert new composition
    const newRows = materials.map(m => manager.create(SupplyPerProductHistory, {
      product_id: productId,
      supply_id: m.supplyId,
      quantity: m.quantity,
      unit_type: m.unitType,
      is_active: true,
      date: new Date(),
    }));
    await manager.save(newRows);
  });
}
```

### Pattern 5: CostsModule as Cross-Cutting Service

**What:** CostsModule imports SuppliesModule (for price queries) and ProductsModule (for composition queries). ProductsModule does NOT import CostsModule. The GET /products endpoint calls CostsService to enrich the product list with calculated costs.

**When to use:** When cost calculation is a derived/read concern that shouldn't pollute the entity write path.

**Trade-offs:** Clear separation. ProductsService handles CRUD. CostsService handles derived calculations. The wiring happens in the controller or a dedicated "read model" service.

**Build order implication:** CostsModule must be built AFTER SuppliesModule and ProductsModule are stable, since it imports both.

## Data Flow

### Flow 1: Authentication — Every Request

```
Browser request (with cookie)
    ↓
Next.js server (extracts JWT from HTTP-only cookie)
    ↓
HTTP request to nemea-back (Authorization: Bearer <jwt>)
    ↓
JwtAuthGuard.canActivate()
    ↓ (if not @Public())
JwtStrategy.validate() → verifies signature → extracts payload (email, sub)
    ↓
AuthService.validateUser(email) → SELECT * FROM users WHERE email = $1
    ↓ (if email not in whitelist → 401 Unauthorized)
    ↓ (if found → attaches { id, email, role } to request.user)
RolesGuard.canActivate() → checks request.user.role vs @Roles() on endpoint
    ↓
Controller handler receives request with request.user populated
```

### Flow 2: GET /products — Cost Calculation

```
GET /products (ADMIN or USER)
    ↓
ProductsController.findAll()
    ↓
ProductsService.findAll() → SELECT active products with catalog joins
    ↓ (returns Product[])
CostsService.enrichWithCosts(products: Product[])
    ↓
  Step A: collect all product_ids
  Step B: SELECT * FROM supplies_per_product_history WHERE product_id IN (...) AND is_active = true
  Step C: collect all unique supply_ids from compositions
  Step D: SELECT DISTINCT ON (supply_id) supply_id, price FROM supplies_price_history ORDER BY supply_id, date DESC
  Step E: in-memory calculation:
            for each product:
              cost = SUM(latest_price[supply_id] × quantity) for each material row
    ↓
ProductsController returns ProductWithCostDto[]
    ↓
Frontend renders table (read-only for USER, editable for ADMIN)
```

### Flow 3: Price Update (creates new history record)

```
ADMIN: POST /supplies/:id/prices { price: 1234.56 }
    ↓
SuppliesController (JwtAuthGuard + @Roles('ADMIN'))
    ↓
SuppliesService.addPrice(id, price)
    ↓
INSERT INTO supplies_price_history (supply_id, price, date) VALUES ($1, $2, NOW())
    ↓ (no UPDATE, old records preserved)
Frontend: next GET /products will automatically use the new price (on-the-fly calc)
```

### Flow 4: Product Composition Change

```
ADMIN: PUT /products/:id/materials [ { supplyId, quantity, unitType } ]
    ↓
ProductsController (RolesGuard → ADMIN only)
    ↓
ProductsService.setMaterials() — opens transaction:
  1. UPDATE supplies_per_product_history SET is_active = false WHERE product_id = $1 AND is_active = true
  2. INSERT new rows (is_active = true, date = NOW())
    ↓
Composition history preserved. Next GET /products reflects new composition.
```

## Module Dependency Graph (Build Order)

```
Phase 2: Backend Scaffold
  AppModule ← (bootstraps everything)

Phase 3: Auth
  AuthModule
    ← UsersRepository (users table)
    ← JwtAuthGuard (global)
    ← RolesGuard (global)
    → exported guards reused by all modules

Phase 4: Supplies
  SuppliesModule
    ← supply, supplier, supply_type, supplies_price_history entities
    → exported SuppliesService (used by CostsModule)

Phase 5: Products
  ProductsModule
    ← product, product_*, supplies_per_product_history, product_price_history entities
    → exported ProductsService (used by CostsModule)

Phase 5 (also): CostsModule
  CostsModule
    ← imports SuppliesModule (price queries)
    ← imports ProductsModule (composition queries)
    → CostsService.enrichWithCosts() called by ProductsController

Phase 6: Config
  TiendanubeConfigModule (avoid naming clash with @nestjs/config)
    ← config_tiendanube entity
```

**Critical dependency:** CostsModule cannot be built until SuppliesModule AND ProductsModule are stable. This means cost calculation must wait until Phase 5 when both supply prices and product compositions exist.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0-100 products / 2-3 users | On-the-fly calculation. No cache. Simple service layer. Current plan. |
| 100-1000 products | Add materialized view or computed `last_cost` column on product, invalidated by triggers. Single query instead of join. |
| 1000+ products / external API consumers | CQRS read model: pre-computed product cost updated by background job when prices change. Redis cache for catalog tables (product_types, etc.). |

### Scaling Priorities (if needed — not for v1)

1. **First bottleneck:** Cost calculation query fan-out. Fix by adding `current_cost DECIMAL` column on product table, recalculated on price/composition change via service event.
2. **Second bottleneck:** Catalog tables (product_type, supply_type) queried on every product load. Fix with in-memory cache (NestJS CacheModule or simple Map with TTL).

## Anti-Patterns

### Anti-Pattern 1: N+1 Cost Calculation

**What people do:** Loop over products, fetch latest price for each supply inside the loop.
```typescript
// WRONG — produces N×M database queries
for (const product of products) {
  for (const material of product.materials) {
    const latestPrice = await this.repo.findOne({ supply_id: material.supply_id, order: { date: 'DESC' } });
    cost += latestPrice.price * material.quantity;
  }
}
```
**Why it's wrong:** 50 products × 3 materials × 1 query each = 150 sequential database roundtrips. Kills response time.
**Do this instead:** Batch-fetch all compositions and all latest prices in 2 queries. Calculate in memory.

### Anti-Pattern 2: synchronize: true in TypeORM

**What people do:** Set `synchronize: true` in TypeORM config for convenience during development.
**Why it's wrong:** Auto-syncs entity changes to DB on every app start. Destroys data in production if entity changes. Hides the real migration diff. Cannot be used on Railway anyway.
**Do this instead:** Always use migrations. `npm run migration:generate -- --name AddSupplyPriceHistory`. Even in dev — it builds the habit and avoids surprises.

### Anti-Pattern 3: Naming ConfigModule to conflict with @nestjs/config

**What people do:** Name the business config module `ConfigModule`, which shadows or conflicts with NestJS's built-in `@nestjs/config` ConfigModule.
**Why it's wrong:** Import confusion, hard-to-debug circular dependency errors, TypeScript type collisions.
**Do this instead:** Name the Tiendanube config module `TiendanubeConfigModule` (file: `tiendanube-config.module.ts`). Import `@nestjs/config`'s ConfigModule explicitly as `NestConfigModule` where needed.

### Anti-Pattern 4: Mutating Price History

**What people do:** UPDATE the latest price record when a price changes instead of inserting a new row.
**Why it's wrong:** Destroys the ability to answer "what did this product cost last month?". Cannot reconstruct historical margins. Eliminates the audit trail that justifies using a history table in the first place.
**Do this instead:** Always INSERT a new row. The table is append-only. Query for the latest with `ORDER BY date DESC LIMIT 1` or `DISTINCT ON (supply_id)`.

### Anti-Pattern 5: Putting Cost Calculation Logic in ProductsService

**What people do:** Add `calculateCost()` to ProductsService because "products have costs".
**Why it's wrong:** ProductsService then imports SuppliesService, creating a fat service that owns both CRUD and derived calculations. Hard to test independently. Violates single responsibility.
**Do this instead:** Keep CostsModule separate. ProductsController calls ProductsService for entity data and CostsService for enrichment. ProductsModule never imports CostsModule.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Google OAuth | NextAuth on frontend handles OAuth dance, returns JWT | Backend only validates the JWT, never talks to Google directly |
| Railway PostgreSQL | TypeORM connection string from env (DATABASE_URL) | SSL required in production (`ssl: { rejectUnauthorized: false }`) |
| Vercel (frontend) | CORS whitelist in main.ts (`origin: process.env.FRONTEND_URL`) | Must be set before any other middleware |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| ProductsController ↔ CostsService | Direct method call (injected service) | CostsModule exports CostsService, imported by ProductsModule |
| AuthGuard ↔ all controllers | NestJS guard chain (request pipeline) | No direct method calls — guards intercept before controller |
| SuppliesModule ↔ CostsModule | CostsModule imports SuppliesModule, calls SuppliesService methods | Unidirectional: costs reads from supplies, never the reverse |

## Sources

- NestJS Guards documentation: https://docs.nestjs.com/guards — MEDIUM confidence (WebFetch unavailable, pattern confirmed via multiple community sources)
- NestJS CQRS recipe: https://docs.nestjs.com/recipes/cqrs — confirmed CQRS is overkill for this scale
- TypeORM Select Query Builder: https://typeorm.io/docs/query-builder/select-query-builder/ — DISTINCT ON support confirmed via WebSearch
- PostgreSQL DISTINCT ON pattern for latest-per-group: standard PostgreSQL — HIGH confidence (well-documented SQL pattern)
- Scope v1 and business rules from `.claude/docs/planning/scope-v1.md` and `business-rules.md` — HIGH confidence (project's own authoritative documents)
- Codebase architecture from `.planning/codebase/ARCHITECTURE.md` — HIGH confidence (reflects actual scaffolded state)
- NestJS layered architecture (controller → service → repository) — HIGH confidence (NestJS official architecture)

---
*Architecture research for: Nemea cost management system (marroquinería)*
*Researched: 2026-02-28*
