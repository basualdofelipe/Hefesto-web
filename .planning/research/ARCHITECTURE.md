# Architecture Research: v1.1 Integration

**Domain:** Tiendanube pricing simulation, investor dashboard, and v1 hardening for an existing NestJS + Next.js cost management app
**Researched:** 2026-03-26
**Confidence:** HIGH (all recommendations based on existing codebase patterns, no new frameworks introduced, dependency analysis verified against actual module wiring)

---

## Current Architecture Snapshot

Before defining v1.1 additions, here is the actual module dependency graph as it exists today. Every recommendation below extends this graph without breaking existing wiring.

```
AppModule
  |-- AuthModule (APP_GUARD: JwtAuthGuard, RolesGuard)
  |     `-- UsersModule
  |-- CatalogsModule
  |-- SuppliersModule
  |-- SuppliesModule
  |-- ProductsModule
  |     `-- CostsModule (imported, uses CostsService for enrichment)
  |-- CostsModule (standalone: imports entity repos directly, NOT other modules)
  |-- ExpensesModule
```

**Key pattern already established:** CostsModule avoids importing SuppliesModule or ProductsModule. Instead, it uses `TypeOrmModule.forFeature([SuppliesPerProductHistory, SupplyPriceHistory])` to access repos directly. This prevents circular dependencies since ProductsModule imports CostsModule.

---

## New Modules for v1.1

### 1. TiendanubeConfigModule (stores editable rates)

**Purpose:** CRUD for Tiendanube configuration data: plans, card rates by withdrawal period, transfer rates, installment fees, IIBB rate, IVA rate.

**Why a separate module:** The naming convention already avoids `ConfigModule` clash with `@nestjs/config`. This is pure admin-editable reference data, similar to CatalogsModule but with a different access pattern (single config set, not a list of items).

**Entities:**

```typescript
// tiendanube-config/entities/tiendanube-plan.entity.ts
@Entity('tiendanube_plans')
export class TiendanubePlan extends BaseEntity {
  @Column({ type: 'varchar', length: 50, unique: true })
  slug!: string; // 'inicial', 'esencial', 'impulso', 'escala'

  @Column({ type: 'varchar', length: 100 })
  label!: string; // Display name

  @Column({ name: 'card_rate_1d', type: 'decimal', precision: 5, scale: 2 })
  cardRate1d!: string; // Tarjeta retiro 1 dia

  @Column({ name: 'card_rate_7d', type: 'decimal', precision: 5, scale: 2 })
  cardRate7d!: string; // Tarjeta retiro 7 dias

  @Column({ name: 'card_rate_14d', type: 'decimal', precision: 5, scale: 2 })
  cardRate14d!: string; // Tarjeta retiro 14 dias

  @Column({ name: 'transfer_rate', type: 'decimal', precision: 5, scale: 2 })
  transferRate!: string; // Transferencia

  @Column({ name: 'tx_tiendanube', type: 'decimal', precision: 5, scale: 2 })
  txTiendanube!: string; // Comision Tiendanube (bonificada con PagoNube)

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}

// tiendanube-config/entities/tiendanube-installment.entity.ts
@Entity('tiendanube_installments')
export class TiendanubeInstallment extends BaseEntity {
  @Column({ type: 'integer' })
  installments!: number; // 1, 3, 6, 9, 12

  @Column({ type: 'decimal', precision: 5, scale: 2 })
  rate!: string; // 0, 8.42, 17.41, 27.04, 37.39

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}

// tiendanube-config/entities/tiendanube-tax-config.entity.ts
@Entity('tiendanube_tax_config')
export class TiendanubeTaxConfig extends BaseEntity {
  @Column({ name: 'iva_rate', type: 'decimal', precision: 5, scale: 2 })
  ivaRate!: string; // 21.00

  @Column({ name: 'iibb_rate', type: 'decimal', precision: 5, scale: 2 })
  iibbRate!: string; // 2.50

  // Singleton row -- enforced by app logic (findOne or upsert)
}
```

**Why 3 tables instead of 1 JSONB:** Plans and installments are independent dimensions that change independently. Plans have rates per withdrawal period. Installments have rates per installment count. Tax config is a singleton. Normalizing them means:
- Each can be edited independently with proper validation
- TypeORM handles types correctly (no JSON parsing)
- Seed data via migrations (like catalogs)
- Future: add historical tracking per rate if needed

**Endpoints:**
```
GET    /tiendanube-config/plans          -- List all plans
PUT    /tiendanube-config/plans/:id      -- Update plan rates (ADMIN)
GET    /tiendanube-config/installments   -- List installment rates
PUT    /tiendanube-config/installments/:id -- Update installment rate (ADMIN)
GET    /tiendanube-config/taxes          -- Get tax config (singleton)
PUT    /tiendanube-config/taxes          -- Update tax config (ADMIN)
GET    /tiendanube-config/all            -- Combined endpoint for calculator
```

**Module wiring:**
```typescript
@Module({
  imports: [TypeOrmModule.forFeature([
    TiendanubePlan,
    TiendanubeInstallment,
    TiendanubeTaxConfig,
  ])],
  controllers: [TiendanubeConfigController],
  providers: [TiendanubeConfigService],
  exports: [TiendanubeConfigService],
})
export class TiendanubeConfigModule {}
```

**Dependencies:** None (standalone reference data, like CatalogsModule).

---

### 2. CalculadoraModule (pricing simulation engine)

**Purpose:** Backend service that implements the forward/inverse Tiendanube pricing calculation using real product costs from CostsService and configurable rates from TiendanubeConfigService.

**Why backend, not frontend-only:** The prototype (`calculadora-tiendanube.jsx`) hardcodes rates and takes cost as manual input. The production version must:
- Pull real product costs from DB (requires CostsService)
- Pull current rates from DB (requires TiendanubeConfigService)
- Be usable by both the calculadora page AND the dashboard/scenarios
- Provide consistent results regardless of frontend state

**Service design:**

```typescript
@Injectable()
export class CalculadoraService {
  constructor(
    private readonly costsService: CostsService,
    private readonly tiendanubeConfigService: TiendanubeConfigService,
  ) {}

  async calcForward(dto: CalcForwardDto): Promise<CalcForwardResult> {
    // dto includes: productId (optional), costoProducto (manual override),
    //   precioVenta, costoEnvio, planSlug, plazoRetiro, medioPago, cuotas
    // If productId provided, fetch cost from CostsService
    // Fetch rates from TiendanubeConfigService
    // Execute the 14-step calculation from business-rules.md
  }

  async calcInverse(dto: CalcInverseDto): Promise<CalcInverseResult> {
    // dto includes: productId (optional), costoProducto (manual override),
    //   gananciaDeseada, costoEnvio, planSlug, plazoRetiro, medioPago, cuotas
    // Binary search using calcForwardInternal()
  }

  async calcBatch(productIds: string[], params: BatchCalcParams): Promise<Map<string, CalcForwardResult>> {
    // For the dashboard: calculate forward for multiple products at once
    // Uses CostsService.calculateAll() for efficiency
  }
}
```

**Critical design decision: CostsService composition.**

The prototype has `costoProducto` as a manual input. The production version needs to support BOTH:
1. **With productId** -- fetches real cost from CostsService automatically
2. **Without productId** -- takes manual `costoProducto` for ad-hoc simulation

This dual-mode prevents the CalculadoraService from being tightly coupled to CostsService for every call, while still supporting the primary use case of calculating margins for actual products.

**Module wiring:**
```typescript
@Module({
  imports: [CostsModule, TiendanubeConfigModule],
  controllers: [CalculadoraController],
  providers: [CalculadoraService],
  exports: [CalculadoraService],
})
export class CalculadoraModule {}
```

**Endpoints:**
```
POST   /calculadora/forward    -- Forward calc (precio -> ganancia)
POST   /calculadora/inverse    -- Inverse calc (ganancia -> precio)
POST   /calculadora/batch      -- Batch forward for multiple products
```

**Dependencies:** CostsModule (for product cost data), TiendanubeConfigModule (for rates).

**No circular dependency risk:** CostsModule does not import CalculadoraModule. CalculadoraModule only reads from CostsModule. The dependency is unidirectional.

---

### 3. ScenariosModule (user-saved pricing scenarios)

**Purpose:** CRUD for investor scenarios. A scenario stores a set of selling-price overrides and Tiendanube config overrides, allowing users to model "what if" pricing without touching real data.

**Entity design:**

```typescript
// scenarios/entities/scenario.entity.ts
@Entity('scenarios')
export class Scenario extends BaseEntity {
  @Column({ type: 'varchar', length: 200 })
  name!: string;

  @Column({ type: 'text', nullable: true })
  description!: string | null;

  @ManyToOne(() => User, { nullable: false })
  @JoinColumn({ name: 'user_id' })
  user!: User;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;

  // Config overrides for the scenario
  @Column({ name: 'plan_slug', type: 'varchar', length: 50, nullable: true })
  planSlug!: string | null;

  @Column({ name: 'plazo_retiro', type: 'varchar', length: 10, nullable: true })
  plazoRetiro!: string | null;

  @Column({ name: 'medio_pago', type: 'varchar', length: 20, nullable: true })
  medioPago!: string | null;

  @Column({ type: 'integer', nullable: true })
  cuotas!: number | null;

  @Column({ name: 'costo_envio', type: 'decimal', precision: 12, scale: 2, nullable: true })
  costoEnvio!: string | null;
}

// scenarios/entities/scenario-price-override.entity.ts
@Entity('scenario_price_overrides')
export class ScenarioPriceOverride extends BaseEntity {
  @ManyToOne(() => Scenario, { nullable: false, onDelete: 'CASCADE' })
  @JoinColumn({ name: 'scenario_id' })
  scenario!: Scenario;

  @ManyToOne(() => Product, { nullable: false })
  @JoinColumn({ name: 'product_id' })
  product!: Product;

  @Column({ name: 'selling_price', type: 'decimal', precision: 12, scale: 2 })
  sellingPrice!: string;
}
```

**Why this shape:**
- **Scenario owns config overrides** (plan, plazo, etc.): Each scenario represents one "situation" -- e.g., "What if we sell on plan Escala, 14-day withdrawal, all card?" These params are scenario-level, not per-product.
- **Price overrides are per product:** Within a scenario, the user sets custom selling prices for specific products. Products without overrides use their real current selling price from `product_price_history`.
- **User scoping:** Each scenario belongs to a user. Investors create their own scenarios. Admin can see all.

**Endpoints:**
```
GET    /scenarios                    -- List user's scenarios (USER gets own, ADMIN gets all)
POST   /scenarios                    -- Create scenario
GET    /scenarios/:id                -- Get scenario with overrides
PUT    /scenarios/:id                -- Update scenario
DELETE /scenarios/:id                -- Soft delete (toggle is_active)
PUT    /scenarios/:id/overrides      -- Set price overrides (bulk upsert)
GET    /scenarios/:id/calculate      -- Run scenario through CalculadoraService (batch)
```

**Module wiring:**
```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([Scenario, ScenarioPriceOverride, Product]),
    CalculadoraModule,
    CostsModule,
  ],
  controllers: [ScenariosController],
  providers: [ScenariosService],
  exports: [ScenariosService],
})
export class ScenariosModule {}
```

**Dependencies:** CalculadoraModule (for `calcBatch`), CostsModule (for real costs), Product entity (FK reference).

---

### 4. DashboardModule (aggregation for investor view)

**Recommendation: Do NOT create a separate DashboardModule.** Instead, add a `DashboardController` to the existing module structure, or implement the dashboard as an endpoint in `ProductsController` or a new thin controller registered in `AppModule`.

**Rationale:**
- The dashboard does not own any entities
- It aggregates data from existing modules: ProductsService, CostsService, CalculadoraService, ScenariosService
- Creating a module for it would add an unnecessary layer

**Recommended approach: Dashboard as a controller-only module:**

```typescript
@Module({
  imports: [ProductsModule, CostsModule, CalculadoraModule, ScenariosModule, TiendanubeConfigModule],
  controllers: [DashboardController],
  providers: [DashboardService],
})
export class DashboardModule {}
```

The DashboardService composes all other services to produce the dashboard payload:

```typescript
@Injectable()
export class DashboardService {
  constructor(
    private readonly productsService: ProductsService,
    private readonly costsService: CostsService,
    private readonly calculadoraService: CalculadoraService,
    private readonly scenariosService: ScenariosService,
  ) {}

  async getDashboardData(userId: string, scenarioId?: string): Promise<DashboardPayload> {
    // 1. Fetch all products with costs (existing flow)
    // 2. If scenarioId: apply price overrides
    // 3. Run calcBatch for all products with scenario config
    // 4. Return aggregated data: products grouped by type with margins
  }
}
```

**Endpoints:**
```
GET    /dashboard                    -- Default view (real prices, real costs)
GET    /dashboard?scenarioId=:id     -- View with scenario overrides
GET    /dashboard/summary            -- Aggregated metrics (total cost, total margin, etc.)
```

---

## Complete v1.1 Module Dependency Graph

```
AppModule
  |-- AuthModule (APP_GUARD: JwtAuthGuard, RolesGuard)
  |     `-- UsersModule
  |-- CatalogsModule                    (existing, no changes)
  |-- SuppliersModule                   (existing, no changes)
  |-- SuppliesModule                    (existing, no changes)
  |-- ProductsModule                    (existing, imports CostsModule)
  |     `-- CostsModule
  |-- CostsModule                       (existing, standalone repos)
  |-- ExpensesModule                    (existing, no changes)
  |-- TiendanubeConfigModule            (NEW, standalone repos)
  |-- CalculadoraModule                 (NEW)
  |     |-- CostsModule
  |     `-- TiendanubeConfigModule
  |-- ScenariosModule                   (NEW)
  |     |-- CalculadoraModule
  |     `-- CostsModule
  |-- DashboardModule                   (NEW, aggregation only)
  |     |-- ProductsModule
  |     |-- CostsModule
  |     |-- CalculadoraModule
  |     |-- ScenariosModule
  |     `-- TiendanubeConfigModule
```

### Dependency DAG (no cycles)

```
TiendanubeConfigModule (0 deps)
     |
     v
CostsModule (0 module deps, uses entity repos)
     |
     v
CalculadoraModule (depends on: CostsModule, TiendanubeConfigModule)
     |
     v
ScenariosModule (depends on: CalculadoraModule, CostsModule)
     |
     v
DashboardModule (depends on: everything above + ProductsModule)
```

**Circular dependency analysis:** No cycles. Every dependency arrow points "downward" in the graph. CostsModule remains the linchpin but continues to use direct repo access (no module-level imports from Products/Supplies), so it stays clean.

---

## New Database Tables

### Migration Plan

**Migration 1: Tiendanube config tables + seed data**
```sql
CREATE TABLE tiendanube_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug VARCHAR(50) UNIQUE NOT NULL,
  label VARCHAR(100) NOT NULL,
  card_rate_1d DECIMAL(5,2) NOT NULL,
  card_rate_7d DECIMAL(5,2) NOT NULL,
  card_rate_14d DECIMAL(5,2) NOT NULL,
  transfer_rate DECIMAL(5,2) NOT NULL,
  tx_tiendanube DECIMAL(5,2) NOT NULL,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tiendanube_installments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  installments INTEGER UNIQUE NOT NULL,
  rate DECIMAL(5,2) NOT NULL,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tiendanube_tax_config (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  iva_rate DECIMAL(5,2) NOT NULL DEFAULT 21.00,
  iibb_rate DECIMAL(5,2) NOT NULL DEFAULT 2.50,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Seed: 4 plans from business-rules.md
INSERT INTO tiendanube_plans (slug, label, card_rate_1d, card_rate_7d, card_rate_14d, transfer_rate, tx_tiendanube)
VALUES
  ('inicial',  'Inicial',  7.69, 5.59, 4.69, 1.50, 2.00),
  ('esencial', 'Esencial', 6.09, 4.39, 3.49, 1.50, 1.50),
  ('impulso',  'Impulso',  5.89, 4.19, 3.29, 0.99, 1.00),
  ('escala',   'Escala',   5.59, 3.89, 2.99, 0.85, 0.70);

-- Seed: 5 installment tiers from business-rules.md
INSERT INTO tiendanube_installments (installments, rate)
VALUES (1, 0), (3, 8.42), (6, 17.41), (9, 27.04), (12, 37.39);

-- Seed: default tax config
INSERT INTO tiendanube_tax_config (iva_rate, iibb_rate) VALUES (21.00, 2.50);
```

**Migration 2: Scenarios tables**
```sql
CREATE TABLE scenarios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(200) NOT NULL,
  description TEXT,
  user_id UUID NOT NULL REFERENCES users(id),
  is_active BOOLEAN DEFAULT true,
  plan_slug VARCHAR(50),
  plazo_retiro VARCHAR(10),
  medio_pago VARCHAR(20),
  cuotas INTEGER,
  costo_envio DECIMAL(12,2),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE scenario_price_overrides (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scenario_id UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  selling_price DECIMAL(12,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(scenario_id, product_id)
);

CREATE INDEX idx_scenario_user ON scenarios(user_id);
CREATE INDEX idx_scenario_overrides_scenario ON scenario_price_overrides(scenario_id);
```

---

## Data Flow: New Features

### Flow 1: Calculadora Forward (product-linked)

```
Frontend: User selects product + Tiendanube config options
  |
  POST /calculadora/forward
    { productId, precioVenta, costoEnvio, planSlug, plazoRetiro, medioPago, cuotas }
  |
  CalculadoraService.calcForward()
    |
    |-- CostsService.calculateForProduct(productId)
    |     Returns: { cost: 6534.48, ... }
    |
    |-- TiendanubeConfigService.getPlanBySlug(planSlug)
    |     Returns: { cardRate14d: 4.69, ... }
    |
    |-- TiendanubeConfigService.getInstallmentRate(cuotas)
    |     Returns: 8.42
    |
    |-- TiendanubeConfigService.getTaxConfig()
    |     Returns: { ivaRate: 21.00, iibbRate: 2.50 }
    |
    |-- Execute 14-step calculation (pure function, no I/O)
    |
    Returns: CalcForwardResult {
      totalCliente, comisionPagoNube, costoFinanciacion,
      ivaDebito, ivaNeto, retencionIIBB, netoRecibido,
      costoProductoConIVA, gananciaReal, margen, ...
    }
```

### Flow 2: Dashboard with Scenario

```
Frontend: Investor selects a saved scenario
  |
  GET /dashboard?scenarioId=abc-123
  |
  DashboardService.getDashboardData(userId, scenarioId)
    |
    |-- ProductsService.findAll() --> all active products
    |-- CostsService.calculateAll() --> cost map
    |-- ScenariosService.findOne(scenarioId) --> scenario + price overrides
    |
    |-- Merge: for each product:
    |     sellingPrice = override ?? product.currentPrice
    |     cost = costMap.get(product.id).cost
    |
    |-- CalculadoraService.calcBatch(products, scenarioConfig)
    |     Uses scenario's plan/plazo/medioPago/cuotas/costoEnvio
    |     Returns: Map<productId, CalcForwardResult>
    |
    Returns: DashboardPayload {
      products: ProductWithMarginData[],
      summary: { totalCost, totalRevenue, totalMargin, avgMarginPercent },
      scenario: { name, config... } | null
    }
```

### Flow 3: Scenario CRUD

```
Investor: Creates a scenario in the dashboard UI
  |
  POST /scenarios
    { name, description, planSlug, plazoRetiro, medioPago, cuotas, costoEnvio }
  |
  Investor: Sets price overrides for specific products
  |
  PUT /scenarios/:id/overrides
    { overrides: [{ productId, sellingPrice }] }
  |
  Investor: Views results
  |
  GET /scenarios/:id/calculate
    |
    ScenariosService:
      1. Load scenario + overrides
      2. Load all products + costs
      3. Apply price overrides
      4. Call calculadoraService.calcBatch()
      5. Return full results
```

---

## Frontend Architecture Changes

### New Pages

```
nemea-front/src/app/
  (app)/
    calculadora/
      page.tsx              -- Calculator page (RSC fetches config + products, client component for interaction)
    dashboard/
      page.tsx              -- Investor dashboard (RSC fetches dashboard data)
    configuracion/
      tiendanube/
        page.tsx            -- Admin: Tiendanube config editor
    admin/
      usuarios/
        page.tsx            -- Admin: User management (backend exists, UI missing)
```

### New Components

```
nemea-front/src/components/
  calculadora/
    CalculadoraClient.tsx   -- Main calculator with forward/inverse toggle
    CalcConfigPanel.tsx     -- Plan/plazo/medioPago/cuotas selectors
    CalcResultPanel.tsx     -- Results display with breakdown
    types.ts
  dashboard/
    DashboardClient.tsx     -- Main dashboard view
    ProductMarginTable.tsx  -- Products grouped by type with margin columns
    DashboardSummary.tsx    -- Total cost, revenue, margin cards
    ScenarioSelector.tsx    -- Dropdown to pick/create scenarios
    ScenarioEditor.tsx      -- Price override editor
    types.ts
  tiendanube-config/
    PlanRatesEditor.tsx     -- Editable table for plan rates
    InstallmentEditor.tsx   -- Editable table for installment rates
    TaxConfigEditor.tsx     -- IVA/IIBB editor
    types.ts
```

### Sidebar Updates

```typescript
// New navigation items
const VENTAS_ITEMS: NavItem[] = [
  { label: 'Calculadora', href: '/calculadora', icon: Calculator },
  { label: 'Dashboard', href: '/dashboard', icon: LayoutDashboard },
];

const CONFIG_ITEMS: NavItem[] = [  // ADMIN only
  { label: 'Config Tiendanube', href: '/configuracion/tiendanube', icon: Settings },
  { label: 'Usuarios', href: '/admin/usuarios', icon: Users },
];
```

### Shared Types (DRY Cleanup)

**Current problem:** `SupplyOption`, `UNIT_LABELS`, `formatDate` are duplicated across multiple files.

**Solution:** Create `nemea-front/src/types/` for shared API response types and `nemea-front/src/lib/formatters.ts` for formatting utilities.

```
nemea-front/src/types/
  api.ts           -- Shared API response types (SupplyOption, CatalogItem, etc.)
  calculadora.ts   -- CalcForwardResult, CalcInverseResult
  dashboard.ts     -- DashboardPayload, ProductWithMarginData
  scenario.ts      -- Scenario, ScenarioPriceOverride

nemea-front/src/lib/
  formatters.ts    -- formatCurrency, formatPercent, formatDate (shared)
```

---

## Hardening: middleware.ts and 401 Handling

### middleware.ts

**Current state:** `proxy.ts` exists with the correct logic but is NOT wired as Next.js middleware. Routes are unprotected on the frontend.

**Fix:** Rename `proxy.ts` to `middleware.ts` at `nemea-front/src/middleware.ts` (Next.js requires this exact name and location). The content is already correct.

### 401 Handling in apiClientFetch

**Current state:** `apiClientFetch` throws a generic error on non-200 responses. JWT expiry causes a 401 that shows as a cryptic error toast.

**Fix:**
```typescript
export async function apiClientFetch<T>(
  path: string,
  token: string,
  options: RequestInit = {},
): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, { /* ... */ });

  if (res.status === 401) {
    // JWT expired or invalidated -- redirect to login
    window.location.href = '/login';
    throw new Error('Session expired');
  }

  if (!res.ok) {
    const error = await res.json().catch(() => ({ message: `Error ${res.status}` }));
    throw new Error(error.message ?? `API error: ${res.status}`);
  }

  return res.json() as Promise<T>;
}
```

---

## Suggested Build Order

Based on the dependency DAG, here is the optimal build order. Each phase can only start after its dependencies are complete.

```
Phase 1: Hardening (no new deps)
  - middleware.ts rename + wiring
  - 401 handling in apiClientFetch
  - Shared types extraction (DRY cleanup)
  - Users admin page (backend exists)

Phase 2: TiendanubeConfigModule (no deps on new modules)
  - Backend: entities + migration + seed + CRUD
  - Frontend: /configuracion/tiendanube admin page

Phase 3: CalculadoraModule (depends on Phase 2 + existing CostsModule)
  - Backend: CalculadoraService + controller
  - Frontend: /calculadora page

Phase 4: ScenariosModule (depends on Phase 3)
  - Backend: entities + migration + CRUD + calculate endpoint
  - Frontend: scenario management in dashboard

Phase 5: DashboardModule (depends on Phases 3 + 4)
  - Backend: DashboardService + controller
  - Frontend: /dashboard page with scenario integration

Phase 6: Product UX improvements (independent, can run in parallel with 2-5)
  - Hierarchical product table grouping
  - BOM group editor scoped to product name
```

**Rationale:**
- Hardening first because it fixes existing bugs and sets up patterns (shared types) used by all new features
- TiendanubeConfig before Calculadora because Calculadora depends on it
- Calculadora before Scenarios because scenarios run calculations
- Dashboard last because it consumes everything
- Product UX is independent and could be parallelized

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Circular Dependency through DashboardModule

**Risk:** DashboardModule imports ProductsModule, which imports CostsModule. If DashboardModule also exports something that ProductsModule needs, we get a cycle.

**Prevention:** DashboardModule is consumer-only. It NEVER exports services. No other module should import DashboardModule.

### Anti-Pattern 2: Putting Calculator Logic in the Frontend

**Risk:** Duplicating the 14-step calculation in both frontend and backend. Frontend uses it for instant previews, backend uses it for scenarios/dashboard.

**Prevention:** The calculation MUST live in the backend (CalculadoraService). Frontend calls the API for every calculation. At this scale (2-3 users, no real-time requirements), API latency is acceptable. If instant preview is needed later, the calculation could be implemented as a shared utility, but for v1.1, keep it backend-only to ensure single source of truth.

### Anti-Pattern 3: Overloading CostsService

**Risk:** Adding calculator-specific methods to CostsService (like "costWithMargin" or "costWithTiendanube"). This pollutes the clean cost-calculation concern with Tiendanube-specific logic.

**Prevention:** CostsService calculates product cost (materials). CalculadoraService calculates Tiendanube pricing. They are separate concerns. CalculadoraService READS from CostsService, never the reverse.

### Anti-Pattern 4: Scenarios Mutating Real Data

**Risk:** Scenario price overrides accidentally updating `product_price_history` instead of `scenario_price_overrides`.

**Prevention:** ScenariosService NEVER writes to `product_price_history` or any table outside `scenarios` and `scenario_price_overrides`. Enforce at the module level by only importing the scenario entity repos.

### Anti-Pattern 5: Fat Dashboard Endpoint

**Risk:** `GET /dashboard` returns everything (products, costs, calculator results, scenarios) in one massive payload that takes seconds to compute.

**Prevention:** Split into focused endpoints:
- `GET /dashboard` -- products with costs and basic margins (fast, no calculator)
- `GET /dashboard?scenarioId=X` -- adds scenario overrides and calculator results (slower, acceptable)
- `GET /dashboard/summary` -- aggregated numbers only (fast)

The frontend can call them progressively (show products first, then enrich with scenario data).

---

## Integration Points Summary

### New Entities (5 tables)

| Table | Module | Relationships |
|-------|--------|---------------|
| `tiendanube_plans` | TiendanubeConfigModule | Standalone |
| `tiendanube_installments` | TiendanubeConfigModule | Standalone |
| `tiendanube_tax_config` | TiendanubeConfigModule | Standalone (singleton) |
| `scenarios` | ScenariosModule | FK to `users` |
| `scenario_price_overrides` | ScenariosModule | FK to `scenarios`, FK to `products` |

### Modified Components (0 entities changed)

No existing entities or tables are modified. All new features are additive.

### New Module to Existing Module Dependencies

| New Module | Imports From | What It Uses |
|------------|-------------|--------------|
| TiendanubeConfigModule | (nothing) | Own repos only |
| CalculadoraModule | CostsModule | `CostsService.calculateForProduct()`, `calculateAll()` |
| CalculadoraModule | TiendanubeConfigModule | `TiendanubeConfigService.getPlanBySlug()`, etc. |
| ScenariosModule | CalculadoraModule | `CalculadoraService.calcBatch()` |
| ScenariosModule | CostsModule | `CostsService.calculateAll()` |
| DashboardModule | ProductsModule | `ProductsService.findAll()` |
| DashboardModule | CostsModule | `CostsService.calculateAll()` |
| DashboardModule | CalculadoraModule | `CalculadoraService.calcBatch()` |
| DashboardModule | ScenariosModule | `ScenariosService.findOne()` |
| DashboardModule | TiendanubeConfigModule | `TiendanubeConfigService.getAll()` |

### Frontend New Routes

| Route | Type | Auth | Purpose |
|-------|------|------|---------|
| `/calculadora` | RSC + Client | ALL | Tiendanube pricing simulator |
| `/dashboard` | RSC + Client | ALL | Investor dashboard with margins |
| `/configuracion/tiendanube` | RSC + Client | ADMIN | Edit Tiendanube rates |
| `/admin/usuarios` | RSC + Client | ADMIN | User management |

---

## Sources

- Codebase analysis: `nemea-back/src/costs/costs.module.ts` -- confirmed CostsModule uses direct repo imports, not module imports (HIGH confidence, source code)
- Codebase analysis: `nemea-back/src/products/products.module.ts` -- confirmed ProductsModule imports CostsModule (HIGH confidence, source code)
- Codebase analysis: `nemea-back/src/app.module.ts` -- confirmed full module registration (HIGH confidence, source code)
- Business rules: `.claude/docs/planning/business-rules.md` -- calcForward/calcInverse algorithms, rate data, test cases (HIGH confidence, first-party)
- Calculator prototype: `_archive/calculadora/calculadora-tiendanube.jsx` -- calcForward/calcInverse implementation reference (HIGH confidence, first-party)
- NestJS module system: NestJS module documentation -- module imports create injection scope, not dependency injection between services directly (HIGH confidence, well-known pattern)
- TypeORM `forFeature` pattern: allows accessing entity repos without importing the owning module -- used by CostsModule to avoid circular deps (HIGH confidence, confirmed in codebase)

---

*Architecture research for: v1.1 Tiendanube pricing, investor dashboard, and hardening*
*Researched: 2026-03-26*
