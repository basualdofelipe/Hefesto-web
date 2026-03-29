# Phase 12: Scenarios - Research

**Researched:** 2026-03-28
**Domain:** What-if pricing simulation with persisted scenarios, built on existing CalculadoraService + TiendanubeConfigModule
**Confidence:** HIGH

## Summary

Phase 12 adds a scenario simulator where investors (and admins) create named what-if scenarios with overridden selling prices and optionally overridden pasarela/plan TN config. Scenarios show recalculated margins using real product costs from the DB combined with the user's hypothetical selling prices, run through the existing `CalculadoraService.calcForward()`.

The technical foundation is solid: `CalculadoraService` already has `calcForward()` (synchronous, pure math) and `calcBatch()` (async, loads config + costs + products). The scenario feature needs a new `ScenariosModule` with 2 new tables (`scenarios`, `scenario_overrides`), a service that loads overrides and feeds them to calcBatch-style logic, and a frontend page at `/escenarios` with a scenario editor including bulk price adjustment.

**Primary recommendation:** Build backend first (entities + migration + CRUD + calculate endpoint), then frontend (list page + editor with product table + bulk override dialog + pasarela/plan selectors). Leverage the existing `CalcBatchItem` interface and `calcForward()` method -- do NOT duplicate calculation logic. The scenario service orchestrates data loading and delegates math to CalculadoraService.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Override scope: price + pasarela + plan (full simulation). Investors override selling price per product, pasarela (gateway) for the scenario, and plan TN for the scenario. Costs stay real.
- Data model: 2 new tables -- scenarios (name, user_id, gateway_slug, plan_id FK to tn_plans, is_public, timestamps) and scenario_overrides (scenario_id FK, product_id FK, override_price decimal, created_at). Cascade delete on scenario removal.
- Visibility: own + public scenarios. isPublic boolean (default false), owner can toggle. GET /scenarios returns own (all) + others' public (read-only). Only owner can edit/delete. Non-public from another user returns 404.
- Bulk override: percentage by type + individual edit. Select product type (or "Todos") + percentage applies to all products of that type. Individual edit per product row. Bulk applies on top of current overrides (not real prices, unless no override exists).
- UI: Dedicated /escenarios page (list + CRUD + editor with product table, bulk actions, pasarela/plan selectors). Dashboard integration deferred to Phase 13.
- Margin calculation: override price if exists, else real price. Pass to CalculadoraService.calcForward(). Show real price, override price, real margin, simulated margin, difference.
- Accessible to ADMIN and USER roles. NOT in ADMIN_ONLY_ROUTES.
- Scenario name required, unique per user (public scenarios from other users don't count for uniqueness).
- Deactivated products: override preserved, shown as "(inactivo)" in UI.
- Bulk override rounds to nearest integer (ARS).

### Claude's Discretion
- Exact entity field names and TypeORM decorators
- DTO validation rules
- Scenario editor UI structure (inline table vs modal vs full page)
- Bulk override UI (dialog/modal or inline controls)
- Difference display (color coding, arrows, etc.)
- API endpoint design (REST routes for scenarios + overrides + calculate)

### Deferred Ideas (OUT OF SCOPE)
- Export scenario to PDF/Excel
- Side-by-side scenario comparison
- Scenario duplication ("fork from existing")
- Cost overrides (simulate supply price changes)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SCEN-01 | User can create a named scenario with overridden selling prices for specific products | ScenariosModule CRUD with scenario + scenario_overrides tables. Entity design follows BaseEntity pattern. Individual override via PUT endpoint. |
| SCEN-02 | User can apply bulk price adjustments (e.g., "+10% all cinturones", "-20% entire catalog") | Frontend bulk override dialog filters products by type, applies percentage math, rounds to integer. Creates/updates scenario_overrides records via bulk upsert endpoint. |
| SCEN-03 | Scenario shows recalculated margins for all affected products using real costs + overridden prices | ScenariosService.calculate() loads scenario overrides, merges with real prices, calls CalculadoraService.calcForward() per product using scenario's gateway/plan config. Returns CalcBatchItem-like results with real vs simulated comparison. |
| SCEN-04 | Scenarios are user-scoped (each investor sees only their own) and persist in DB | scenarios.user_id FK to users. Service filters by userId from JWT. is_public toggle enables sharing. Querying non-owned non-public scenario returns 404. |
</phase_requirements>

## Standard Stack

### Core (already installed -- zero new dependencies)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Backend framework | Existing project stack |
| TypeORM | 0.3.x | ORM + migrations | Existing project stack, entity patterns established |
| PostgreSQL | 15+ | Database | Existing project stack, UUID generation, DISTINCT ON support |
| Next.js | 16.x | Frontend framework | Existing project stack, App Router |
| Shadcn/ui | latest | UI components | Existing project stack, all needed components installed |
| class-validator | installed | DTO validation | Existing pattern across all DTOs |
| class-transformer | installed | DTO transformation | Existing pattern |

### Supporting (already installed)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| lucide-react | installed | Icons | Sidebar icon for Escenarios, UI elements |
| sonner | installed | Toast notifications | Success/error feedback on CRUD operations |
| @nestjs/swagger | installed | API docs | Swagger decorators on new endpoints |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Separate scenario_overrides table | JSONB column on scenarios | JSONB loses FK integrity to products, harder to query individual overrides, harder to validate. Separate table matches project pattern. |
| Bulk override in backend | Frontend-only calculation + batch save | Backend bulk would add complexity; frontend can calculate percentages and send final override values. Backend stays a simple upsert. |

**Installation:** None required. Zero new npm dependencies (per v1.1 decision).

## Architecture Patterns

### Recommended Module Structure

```
nemea-back/src/scenarios/
  scenarios.module.ts
  scenarios.service.ts
  scenarios.controller.ts
  entities/
    scenario.entity.ts
    scenario-override.entity.ts
  dto/
    create-scenario.dto.ts
    update-scenario.dto.ts
    upsert-overrides.dto.ts
    calculate-scenario.dto.ts
    scenario-response.dto.ts

nemea-front/src/app/(app)/escenarios/
  page.tsx                          # RSC: fetch scenarios list
  [id]/
    page.tsx                        # RSC: fetch scenario detail + products + config
    ScenarioEditorClient.tsx        # Client: full editor with product table

nemea-front/src/components/scenarios/
  types.ts                          # Scenario, ScenarioOverride interfaces
  ScenarioListClient.tsx            # Client: list with create/delete
  ScenarioCard.tsx                  # Individual scenario card in list
  ProductOverrideTable.tsx          # Editable product table with override prices
  BulkOverrideDialog.tsx            # Dialog for "+X% to type Y"
  GatewayPlanSelector.tsx           # Reuse pattern from calculadora GatewaySelectors
  MarginComparisonRow.tsx           # Row showing real vs simulated margins
```

### Pattern 1: ScenariosModule Dependency Wiring

**What:** ScenariosModule imports CalculadoraModule (for calcForward), CostsModule (for real costs), ProductsModule (for product data), TiendanubeConfigModule (for config in calcForward)
**When to use:** Always -- this is the required module graph

```typescript
// Source: codebase analysis of existing module patterns
@Module({
  imports: [
    TypeOrmModule.forFeature([Scenario, ScenarioOverride]),
    CalculadoraModule,    // for CalculadoraService.calcForward()
    CostsModule,          // for CostsService.calculateAll()
    ProductsModule,       // for ProductsService.findAll()
    TiendanubeConfigModule, // for TiendanubeConfigService.getAll()
  ],
  controllers: [ScenariosController],
  providers: [ScenariosService],
  exports: [ScenariosService], // for Phase 13 DashboardModule
})
export class ScenariosModule {}
```

**No circular dependency risk:** ScenariosModule only reads from all imported modules. No other module imports ScenariosModule (until Phase 13).

### Pattern 2: Scenario Calculate Flow (reuse calcForward, not calcBatch)

**What:** The scenario calculate endpoint cannot directly use `calcBatch()` because calcBatch loads real prices from products. Scenarios need to inject override prices. Instead, the flow is:
1. Load TN config once via `tiendanubeConfigService.getAll()`
2. Load all product costs via `costsService.calculateAll()`
3. Load all products via `productsService.findAll()`
4. Load scenario + overrides
5. For each product: determine effective price (override or real), call `calculadoraService.calcForward()` synchronously
6. Return enriched results

**Why not modify calcBatch:** calcBatch is a focused method with a clear contract. Adding scenario override logic to it pollutes its single responsibility. Better to orchestrate at the ScenariosService level.

```typescript
// Pattern: ScenariosService.calculate()
async calculate(scenarioId: string, userId: string): Promise<ScenarioCalcResult> {
  const scenario = await this.findOneOrFail(scenarioId, userId);
  const overrides = await this.overrideRepo.find({
    where: { scenario: { id: scenarioId } },
  });
  const overrideMap = new Map(overrides.map(o => [o.product.id, parseFloat(o.overridePrice as string)]));

  const config = await this.tiendanubeConfigService.getAll();
  const costMap = await this.costsService.calculateAll();
  const products = await this.productsService.findAll();

  // Resolve gateway config: scenario overrides or defaults
  const gatewaySlug = scenario.gatewaySlug ?? 'pago_nube';
  // ... resolve paymentMethod, withdrawalDays from config defaults or scenario

  const results: ScenarioProductResult[] = [];
  for (const product of products) {
    const cost = costMap.get(product.id)?.cost ?? 0;
    const realPrice = product.currentPrice ? parseFloat(product.currentPrice as string) : null;
    const overridePrice = overrideMap.get(product.id) ?? null;
    const effectivePrice = overridePrice ?? realPrice;

    let simResult: CalcResult | null = null;
    let realResult: CalcResult | null = null;

    if (effectivePrice !== null && effectivePrice > 0) {
      simResult = this.calculadoraService.calcForward({
        precioVenta: effectivePrice,
        costoEnvio: 0,
        costoProducto: cost,
        gatewaySlug,
        paymentMethod,
        withdrawalDays,
        installments,
        planSlug: scenario.planSlug,
        config,
      });
    }

    if (realPrice !== null && realPrice > 0) {
      // Use default config (not scenario overrides) for "real" comparison
      realResult = this.calculadoraService.calcForward({
        precioVenta: realPrice,
        costoEnvio: 0,
        costoProducto: cost,
        gatewaySlug: defaultGatewaySlug,
        paymentMethod: defaultPaymentMethod,
        withdrawalDays: defaultWithdrawalDays,
        installments: defaultInstallments,
        planSlug: defaultPlanSlug,
        config,
      });
    }

    results.push({
      productId: product.id,
      productName: buildProductName(product),
      cost,
      realPrice,
      overridePrice,
      effectivePrice,
      simResult,
      realResult,
      isActive: product.isActive,
    });
  }

  return { scenario, results };
}
```

### Pattern 3: User-Scoping with JWT User Extraction

**What:** Every scenario endpoint extracts the userId from the JWT to scope queries.
**When to use:** All scenario CRUD and calculate operations.

```typescript
// Pattern: Extract user from request (matches existing auth patterns)
@Get()
@Roles(Role.ADMIN, Role.USER)
async findAll(@Req() req: Request): Promise<Scenario[]> {
  const userId = req.user.id; // from JwtAuthGuard
  return this.scenariosService.findAll(userId);
}
```

The findAll query returns: own scenarios (WHERE user_id = :userId) UNION public scenarios from others (WHERE is_public = true AND user_id != :userId). Use QueryBuilder or two queries merged.

### Pattern 4: Bulk Override as Frontend Math + Backend Upsert

**What:** Bulk override calculation happens in the frontend (user selects type + percentage, frontend computes new prices), then sends the computed overrides to backend as a batch upsert.
**Why:** The backend doesn't need to know about "percentages." It just receives concrete override prices. This keeps the API simple and makes the UI responsive (user sees the computed prices before saving).

```typescript
// Frontend: compute overrides from percentage
function applyBulkOverride(
  products: ProductWithPrice[],
  filterTypeId: string | 'all',
  percentage: number,
  existingOverrides: Map<string, number>,
): Map<string, number> {
  const newOverrides = new Map(existingOverrides);
  for (const product of products) {
    if (filterTypeId !== 'all' && product.type.id !== filterTypeId) continue;
    const basePrice = existingOverrides.get(product.id) ?? product.currentPrice;
    if (basePrice === null) continue;
    const newPrice = Math.round(basePrice * (1 + percentage / 100));
    newOverrides.set(product.id, newPrice);
  }
  return newOverrides;
}
```

### Anti-Patterns to Avoid

- **Writing to product_price_history:** Scenarios MUST never touch this table. ScenariosService only writes to `scenarios` and `scenario_overrides`. Enforced at module level by NOT importing ProductPriceHistory repo.
- **Duplicating calcForward logic:** All margin calculation goes through `CalculadoraService.calcForward()`. Never reimplement the 14-step formula.
- **N+1 queries in calculate:** Load config, costs, products, and overrides ONCE each. Do NOT query per-product in a loop. The calcForward call is synchronous pure math -- no DB calls inside the loop.
- **Fat scenario entity with embedded overrides:** Overrides belong in a separate table with FK to products. Using JSONB would lose referential integrity.
- **Modifying calcBatch for scenario support:** Keep calcBatch clean for its original purpose (Phase 13 dashboard without scenarios). Build scenario calculation as a separate method in ScenariosService.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Margin calculation | Custom formula in scenario service | CalculadoraService.calcForward() | Formula has 14 steps with IVA/IIBB/CPT; single source of truth already exists and is tested |
| Product cost calculation | Custom cost query in scenario service | CostsService.calculateAll() | BOM + supply price resolution already handled with DISTINCT ON latest prices |
| TN config resolution | Load config manually from repos | TiendanubeConfigService.getAll() | Already handles raw SQL, decimal parsing, rate resolution |
| UUID generation | Custom ID generation | TypeORM BaseEntity pattern | Project convention: @PrimaryGeneratedColumn('uuid') |
| Input validation | Manual checks in controller | class-validator decorators on DTOs | Project pattern, consistent with all existing endpoints |
| Toast notifications | Custom notification system | sonner toast | Already installed and used across all CRUD operations |

**Key insight:** Phase 12 is primarily an orchestration layer. The hard math (calcForward), data loading (CostsService, ProductsService, TiendanubeConfigService), and UI primitives (Shadcn table, dialog, select, input) all exist. The new code is: entity definitions, CRUD endpoints, data merging logic, and a scenario editor UI.

## Common Pitfalls

### Pitfall 1: Scenario Data Isolation Violation
**What goes wrong:** Accidentally writing override prices to `product_price_history` instead of `scenario_overrides`, polluting real pricing data.
**Why it happens:** The existing pattern for price changes is `productsService.addPrice()` which writes to product_price_history. Copy-paste from existing code could use the wrong service.
**How to avoid:** ScenariosModule only imports TypeOrmModule.forFeature([Scenario, ScenarioOverride]). It does NOT import Product or ProductPriceHistory repos directly. All product data comes through ProductsService (read-only). The ScenarioOverride entity has a separate table with its own lifecycle.
**Warning signs:** If you see `priceHistoryRepo` anywhere in scenarios code, it's wrong.

### Pitfall 2: is_public Visibility Logic Errors
**What goes wrong:** User A can see User B's non-public scenarios, or the uniqueness constraint on scenario names fails because it includes other users' public scenarios.
**Why it happens:** The visibility rule is nuanced: own (all) + others' public (read-only). Name uniqueness is per-user only. These are two different query shapes.
**How to avoid:**
- findAll: `WHERE user_id = :userId OR (is_public = true AND user_id != :userId)`
- findOne: `WHERE id = :id AND (user_id = :userId OR is_public = true)`
- name uniqueness: `WHERE name = :name AND user_id = :userId` (ignore other users entirely)
- edit/delete: `WHERE id = :id AND user_id = :userId` (owner only, regardless of is_public)
**Warning signs:** A user can edit someone else's public scenario, or name validation rejects names that another user has used.

### Pitfall 3: Bulk Override Applying to Real Prices Instead of Current Overrides
**What goes wrong:** CONTEXT.md specifies: "Bulk adjustments apply ON TOP of current overrides (not on top of real prices, unless no override exists yet)." If the frontend always uses the real product price as the base for percentage calculations, sequential bulk operations compound incorrectly.
**Why it happens:** The natural implementation is `product.currentPrice * (1 + percentage/100)`. But if the user already overrode Billetera Hefesto to $95,000 (from $87,000), then "+10%" should yield $104,500, not $95,700.
**How to avoid:** The bulk override function must check the override map first: `basePrice = existingOverrides.get(product.id) ?? product.currentPrice`. This is already shown in the architecture pattern above.
**Warning signs:** User applies "+10%" twice and gets 121% of real price instead of 110% of the first override.

### Pitfall 4: TypeORM Decimal Columns Returned as Strings
**What goes wrong:** `scenario_overrides.override_price` is `decimal(12,2)` which TypeORM returns as string. If the scenario calculate flow passes this string directly to `calcForward()` without parseFloat, arithmetic fails silently in JavaScript (string concatenation instead of addition).
**Why it happens:** TypeORM decimal behavior (documented in issue #2937). The project already handles this in multiple places (CostsService, TiendanubeConfigService, CalculadoraService).
**How to avoid:** Parse all decimal fields at the service boundary. The entity types `overridePrice` as `string` (matching TypeORM convention). The service method `parseFloat(override.overridePrice as string)` converts before use.
**Warning signs:** Override prices display correctly but margin calculations are wrong or NaN.

### Pitfall 5: Default Gateway/Plan Config for "Real" Margin Comparison
**What goes wrong:** The scenario editor shows "real margin" vs "simulated margin." But what gateway/plan config should the "real" column use? If there's no system-wide default, the comparison is meaningless.
**Why it happens:** The current calculadora requires the user to select gateway/plan/method every time. There's no concept of a "default" TN configuration.
**How to avoid:** Two approaches (recommend approach 1):
1. The "real margin" column uses the scenario's own gateway/plan config but with real prices. This way the user is comparing "price difference" only, not "price + config difference."
2. Alternative: skip the real margin column entirely and show only cost, real price, override price, and simulated margin. The user mentally compares.
**Warning signs:** Users confused by what "real margin" means if it uses different config than the scenario.

## Code Examples

### Entity: Scenario (follows BaseEntity pattern)

```typescript
// Source: codebase analysis of user.entity.ts + product.entity.ts patterns
@Entity('scenarios')
export class Scenario extends BaseEntity {
  @Column({ type: 'varchar', length: 200, nullable: false })
  name!: string;

  @ManyToOne(() => User, { nullable: false })
  @JoinColumn({ name: 'user_id' })
  user!: User;

  @Column({ name: 'is_public', type: 'boolean', default: false })
  isPublic!: boolean;

  @Column({ name: 'gateway_slug', type: 'varchar', length: 50, nullable: true })
  gatewaySlug!: string | null;

  @Column({ name: 'payment_method', type: 'varchar', length: 50, nullable: true })
  paymentMethod!: string | null;

  @Column({ name: 'withdrawal_days', type: 'integer', nullable: true })
  withdrawalDays!: number | null;

  @Column({ type: 'integer', nullable: true })
  installments!: number | null;

  @ManyToOne(() => TnPlan, { nullable: true })
  @JoinColumn({ name: 'plan_id' })
  plan!: TnPlan | null;

  @OneToMany(() => ScenarioOverride, (o) => o.scenario, { cascade: true })
  overrides!: ScenarioOverride[];
}
```

### Entity: ScenarioOverride

```typescript
// Source: codebase analysis of supplies-per-product-history.entity.ts pattern
@Entity('scenario_overrides')
@Unique(['scenario', 'product'])
export class ScenarioOverride extends BaseEntity {
  @ManyToOne(() => Scenario, (s) => s.overrides, {
    nullable: false,
    onDelete: 'CASCADE',
  })
  @JoinColumn({ name: 'scenario_id' })
  scenario!: Scenario;

  @ManyToOne(() => Product, { nullable: false, eager: true })
  @JoinColumn({ name: 'product_id' })
  product!: Product;

  @Column({
    name: 'override_price',
    type: 'decimal',
    precision: 12,
    scale: 2,
    nullable: false,
  })
  overridePrice!: string; // TypeORM returns decimal as string
}
```

### Migration Pattern (follows existing convention)

```typescript
// Source: 1772800000000-CreateTiendanubeConfigTables.ts pattern
export class CreateScenariosTables1772900000000 implements MigrationInterface {
  name = 'CreateScenariosTables1772900000000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `CREATE TABLE "scenarios" (
        "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
        "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
        "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
        "name" character varying(200) NOT NULL,
        "user_id" uuid NOT NULL,
        "is_public" boolean NOT NULL DEFAULT false,
        "gateway_slug" character varying(50),
        "payment_method" character varying(50),
        "withdrawal_days" integer,
        "installments" integer,
        "plan_id" uuid,
        CONSTRAINT "PK_scenarios_id" PRIMARY KEY ("id"),
        CONSTRAINT "FK_scenarios_user" FOREIGN KEY ("user_id") REFERENCES "users"("id"),
        CONSTRAINT "FK_scenarios_plan" FOREIGN KEY ("plan_id") REFERENCES "tn_plans"("id")
      )`
    );

    await queryRunner.query(
      `CREATE INDEX "IDX_scenarios_user" ON "scenarios" ("user_id")`
    );

    await queryRunner.query(
      `CREATE TABLE "scenario_overrides" (
        "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
        "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
        "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
        "scenario_id" uuid NOT NULL,
        "product_id" uuid NOT NULL,
        "override_price" decimal(12,2) NOT NULL,
        CONSTRAINT "PK_scenario_overrides_id" PRIMARY KEY ("id"),
        CONSTRAINT "FK_scenario_overrides_scenario" FOREIGN KEY ("scenario_id") REFERENCES "scenarios"("id") ON DELETE CASCADE,
        CONSTRAINT "FK_scenario_overrides_product" FOREIGN KEY ("product_id") REFERENCES "products"("id"),
        CONSTRAINT "UQ_scenario_overrides_scenario_product" UNIQUE ("scenario_id", "product_id")
      )`
    );

    await queryRunner.query(
      `CREATE INDEX "IDX_scenario_overrides_scenario" ON "scenario_overrides" ("scenario_id")`
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE "scenario_overrides"`);
    await queryRunner.query(`DROP TABLE "scenarios"`);
  }
}
```

### API Endpoint Design

```
POST   /scenarios                    -- Create scenario (name, gatewaySlug, planId, etc.)
GET    /scenarios                    -- List: own + public from others
GET    /scenarios/:id                -- Get scenario with overrides
PUT    /scenarios/:id                -- Update scenario metadata
DELETE /scenarios/:id                -- Delete scenario (hard delete with cascade)
PUT    /scenarios/:id/overrides      -- Bulk upsert overrides [{productId, overridePrice}]
GET    /scenarios/:id/calculate      -- Run scenario through CalculadoraService
PATCH  /scenarios/:id/toggle-public  -- Toggle isPublic
```

### Frontend: RSC + Client Component Pattern (matches calculadora)

```typescript
// Source: nemea-front/src/app/(app)/calculadora/page.tsx pattern
// escenarios/[id]/page.tsx
export default async function ScenarioEditorPage({
  params,
}: { params: Promise<{ id: string }> }): Promise<ReactElement> {
  const { id } = await params;
  const [scenarioRes, productsRes, configRes] = await Promise.all([
    apiFetch<{ data: ScenarioWithOverrides }>(`/api/scenarios/${id}`),
    apiFetch<{ data: Product[] }>('/api/products'),
    apiFetch<{ data: TiendanubeConfigAll }>('/api/tiendanube-config/all'),
  ]);

  return (
    <ScenarioEditorClient
      scenario={scenarioRes.data}
      products={productsRes.data}
      config={configRes.data}
    />
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hardcoded TN rates in prototype | Admin-editable rates in DB (Phase 10) | v1.1 Phase 10 | Scenarios use live rates from TiendanubeConfigService |
| Manual cost entry in calculadora | Auto-fetched from CostsService | v1.1 Phase 11 | Scenarios get real costs automatically |
| No scenario persistence | Separate scenario tables with FK integrity | Phase 12 (this phase) | Scenarios persist across sessions, scoped per user |

**Existing infrastructure this phase leverages:**
- `CalculadoraService.calcForward()` -- proven 14-step formula with tests
- `CostsService.calculateAll()` -- efficient 2-query bulk cost calculation
- `ProductsService.findAll()` -- products with eager-loaded dimensions
- `TiendanubeConfigService.getAll()` -- full TN config in one call with decimal parsing
- Shadcn Dialog, Table, Input, Select, Switch, Badge, Card -- all installed
- sonner toast -- installed and used across the app
- apiClientFetch / apiFetch -- client/server API wrappers with 401 handling

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (via @nestjs/testing) |
| Config file | nemea-back/jest config in package.json |
| Quick run command | `cd nemea-back && npx jest --testPathPattern=scenarios --no-coverage` |
| Full suite command | `cd nemea-back && npx jest --no-coverage` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SCEN-01 | Create scenario + set overrides | unit | `npx jest scenarios.service.spec.ts --no-coverage` | Wave 0 |
| SCEN-02 | Bulk override applies percentage correctly | unit (frontend logic) | Manual verification -- frontend math utility | Wave 0 |
| SCEN-03 | Calculate returns real vs simulated margins | unit | `npx jest scenarios.service.spec.ts --no-coverage` | Wave 0 |
| SCEN-04 | User scoping -- own + public visibility | unit | `npx jest scenarios.service.spec.ts --no-coverage` | Wave 0 |

### Sampling Rate
- **Per task commit:** `cd nemea-back && npx jest --testPathPattern=scenarios --no-coverage`
- **Per wave merge:** `cd nemea-back && npx jest --no-coverage`
- **Phase gate:** Full suite green before /gsd:verify-work

### Wave 0 Gaps
- [ ] `nemea-back/src/scenarios/scenarios.service.spec.ts` -- covers SCEN-01, SCEN-03, SCEN-04
- [ ] Frontend bulk override utility test (optional -- simple math function)

## Open Questions

1. **Default gateway/plan for "real" margin column**
   - What we know: Scenarios override gateway/plan. The CONTEXT.md says to show "real margin" and "simulated margin."
   - What's unclear: What gateway/plan config should the "real margin" use? There's no system-wide "default gateway" concept.
   - Recommendation: Use the scenario's own gateway/plan for both columns. The comparison isolates price differences only. Alternatively, the "real margin" column could simply show gross margin (price - cost) without TN deductions, since the user hasn't selected a config for "real."

2. **Scenario editor: when to trigger calculate**
   - What we know: The editor shows a product table with override prices and margins.
   - What's unclear: Should calculation happen on every keystroke (expensive API call per product) or on explicit "Calcular" button?
   - Recommendation: Calculate on explicit action (save + calculate button). The initial load fetches results via GET /scenarios/:id/calculate. Individual price edits are local state until saved.

3. **Plan FK vs plan_slug in scenarios table**
   - What we know: CONTEXT.md says "plan_id (override, FK to tn_plans)". But the calculadora uses `planSlug` strings throughout.
   - What's unclear: Should scenarios store the plan as FK (id) or slug?
   - Recommendation: Store as FK (plan_id UUID references tn_plans). Resolve to slug in the service when calling calcForward. This ensures referential integrity and handles plan deletions gracefully.

## Environment Availability

Step 2.6: SKIPPED (no external dependencies identified). Phase 12 uses only existing stack: NestJS, TypeORM, PostgreSQL, Next.js, Shadcn/ui. All verified available from previous phases.

## Project Constraints (from CLAUDE.md)

- **TypeScript strict** -- no `any`, explicit return types on all functions
- **Single quotes** for strings
- Run `npm run lint` and `npm run prettier:fix` before commits
- Never commit without explicit user request
- Never use `git add -A` without reviewing staged files
- Include `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` in commits
- UUIDs on all entities (BaseEntity pattern with `@PrimaryGeneratedColumn('uuid')`)
- Branching strategy: phase branch in sub-repos, never commit directly to development
- Zero new npm dependencies for v1.1

## Sources

### Primary (HIGH confidence)
- Codebase: `nemea-back/src/calculadora/calculadora.service.ts` -- calcForward and calcBatch implementations, CalcForwardParams interface
- Codebase: `nemea-back/src/calculadora/dto/calc-result.dto.ts` -- CalcResult, CalcBatchItem, CalcError interfaces
- Codebase: `nemea-back/src/tiendanube-config/tiendanube-config.service.ts` -- TiendanubeConfigAll interface, getAll() method, decimal parsing pattern
- Codebase: `nemea-back/src/common/entities/base.entity.ts` -- BaseEntity pattern (UUID + timestamps)
- Codebase: `nemea-back/src/users/entities/user.entity.ts` -- User entity (FK target for scenarios)
- Codebase: `nemea-back/src/products/entities/product.entity.ts` -- Product entity (FK target for overrides)
- Codebase: `nemea-back/src/database/migrations/1772800000000-CreateTiendanubeConfigTables.ts` -- migration pattern
- Codebase: `nemea-front/src/middleware.ts` -- ADMIN_ONLY_ROUTES list (escenarios must NOT be included)
- Codebase: `nemea-front/src/components/layout/AppSidebar.tsx` -- sidebar structure, NavItem pattern, Herramientas group
- Codebase: `nemea-front/src/app/(app)/calculadora/page.tsx` -- RSC + client component pattern
- Codebase: `nemea-front/src/components/tiendanube-config/types.ts` -- TiendanubeConfigAll frontend type
- Codebase: `nemea-front/src/components/products/types.ts` -- Product frontend type, formatting helpers
- Planning: `.planning/phases/12-scenarios/12-CONTEXT.md` -- all locked decisions
- Planning: `.planning/research/ARCHITECTURE.md` -- ScenariosModule design, dependency DAG
- Planning: `.planning/research/PITFALLS.md` -- Pitfall 4 (scenario data isolation)
- Planning: `.planning/research/FEATURES.md` -- scenario simulator feature analysis

### Secondary (MEDIUM confidence)
- Planning: `.planning/phases/11-calculadora/11-CONTEXT.md` -- CalculadoraService API contract decisions
- Planning: `.planning/phases/11-calculadora/11-01-SUMMARY.md` -- calcBatch implementation details

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- zero new dependencies, all existing patterns
- Architecture: HIGH -- module wiring follows established DAG, entity design follows BaseEntity convention, calculation reuses proven calcForward
- Pitfalls: HIGH -- all pitfalls grounded in actual codebase patterns (decimal strings, data isolation, visibility logic)
- Code examples: HIGH -- all derived from existing codebase patterns, not hypothetical

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable -- no external dependencies, all patterns from existing codebase)
