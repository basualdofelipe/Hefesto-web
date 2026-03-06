# Phase 6: Cost Calculation - Research

**Researched:** 2026-03-06
**Domain:** Dynamic cost calculation, batched SQL queries, NestJS module composition, Next.js product detail page
**Confidence:** HIGH

## Summary

Phase 6 is the core value of the app: calculating product costs dynamically from latest supply prices and displaying them in the product list and a new product detail page. The codebase already has all the foundational patterns needed -- the DISTINCT ON batch query for latest prices exists in both `SuppliesService.findAll()` and `ProductsService.findAll()`. The BOM entity (`SuppliesPerProductHistory`) eagerly loads supply data. The work is primarily composing existing patterns into a new `CostsModule` and extending the frontend with cost/margin columns plus a detail page.

No new libraries are needed. No migrations are needed. The data model is complete from Phase 5. This phase is pure business logic (service-level calculation) + API response enrichment + frontend display.

**Primary recommendation:** Create a `CostsModule` with `CostsService` that performs exactly 2 SQL queries (active BOM entries + DISTINCT ON latest supply prices), computes costs in memory, and exposes the calculation through a modified `GET /products` response. Frontend adds cost/margin columns to the existing table and creates a new `/productos/:id` detail page.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Cost & margin columns in product list: Costo, Precio Venta, Margen ($ and %) columns. Margen % = (Margen / Costo) x 100. All values sin IVA.
- Column order: SKU | Nombre | Terminacion | Color | Talle | Costo | Precio Venta | Margen
- Cost breakdown in expanded row: Enrich BOM table with Precio Unitario and Costo Linea columns, total row, margin summary below.
- Product detail page at `/productos/:id` with header + BOM with cost breakdown + margin summary. Back navigation. Reuses same modal components.
- On-the-fly calculation per request -- no cache, no background job, no WebSocket, no polling.
- 2 SQL queries + in-memory calculation: 1) active BOM entries for all products, 2) DISTINCT ON latest price per supply.
- CostsModule separate from ProductsModule (imports both SuppliesModule + ProductsModule).
- Auto-refresh on navigation -- product list fetches fresh data on page load.
- Group header aggregation: type group headers show count + average cost. Format: "Billeteras (4 productos) -- Costo prom: $4.800". Average excludes products without BOM.
- Edge cases: Product without BOM shows "--" with tooltip. Supply without price shows partial cost with warning. Inactive supply in BOM included in cost (dimmed with warning). Product without selling price shows "--" in precio and margin.

### Claude's Discretion
- Exact styling of cost/margin columns (color coding, font weight)
- Warning indicator design (icon choice, tooltip implementation)
- Product detail page exact spacing and card styling
- Loading states for cost calculation
- How average cost is displayed in group headers (font size, color)
- BOM table styling in product detail page vs expand inline

### Deferred Ideas (OUT OF SCOPE)
- Calculadora Tiendanube with ganancia min/max range by payment method and shipping -- Phase 7
- IVA on supply costs (credito fiscal) and IVA on selling price -- Phase 7
- IIBB deduction on sales -- Phase 7
- Comisiones MercadoPago/Tiendanube by payment method -- Phase 7
- BOM version history UI (old compositions) -- future phase
- Cost trend over time -- v2
- Export cost data to Excel/PDF -- v2
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| COST-01 | System calculates product cost dynamically: SUM(latest_price(supply) x quantity) for all active materials | CostsService with 2-query DISTINCT ON pattern already proven in SuppliesService and ProductsService. In-memory map join and multiplication. |
| COST-02 | Product list shows calculated cost per product | Extend GET /products response with `cost`, `costBreakdown`, `hasWarnings` fields. Frontend adds Costo and Margen columns to ProductTypeGroup table. |
| COST-03 | When a supply price is updated, all products using that supply reflect the new cost automatically | On-the-fly calculation guarantees this -- no cache to invalidate. Navigation triggers fresh server-component fetch. |
| COST-04 | Product detail shows cost breakdown by material | New `/productos/:id` page with enriched BOM table (Precio Unitario, Costo Linea columns). Single-product endpoint or reuse list data. |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Backend framework | Already in use |
| TypeORM | 0.3.28 | ORM + raw SQL via `.query()` | Already in use, DISTINCT ON pattern proven |
| Next.js | 16.x | Frontend framework | Already in use, App Router with server components |
| Shadcn/ui | latest | UI components (Table, Badge, Tooltip) | Already in use across all pages |
| Tailwind v4 | latest | Styling | Already in use |

### Supporting (already installed)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| lucide-react | latest | Icons (AlertTriangle, Info, ArrowLeft) | Warning indicators, navigation |
| sonner | latest | Toast notifications | Error feedback |
| next-auth | latest | Session/role checking | isAdmin guard on detail page |

### No New Dependencies
This phase requires zero new npm packages. Everything needed is already installed.

## Architecture Patterns

### Backend: CostsModule Structure
```
src/
├── costs/
│   ├── costs.module.ts          # Imports ProductsModule + SuppliesModule
│   ├── costs.service.ts         # 2-query + in-memory calc
│   └── dto/
│       └── product-with-cost.dto.ts  # Response shape
├── products/
│   ├── products.controller.ts   # Modified GET /products to use CostsService
│   └── products.module.ts       # Already exports ProductsService
└── supplies/
    └── supplies.module.ts       # Already exports SuppliesService
```

### Pattern 1: 2-Query Batch Cost Calculation
**What:** Fetch all active BOM entries in one query, fetch DISTINCT ON latest supply prices in one query, join in memory.
**When to use:** Always -- this is the only calculation method (locked decision).
**Example:**
```typescript
// Query 1: All active BOM entries for all products (or specific products)
const bomEntries = await this.bomRepo.find({
  where: { isActive: true },
  relations: ['supply'],
});

// Query 2: Latest price per supply using DISTINCT ON
const latestPrices: { supply_id: string; price: string }[] =
  await this.priceHistoryRepo.query(
    `SELECT DISTINCT ON (supply_id) supply_id, price
     FROM supply_price_history
     ORDER BY supply_id, created_at DESC`,
  );

// In-memory: build supplyId -> price map, then compute per product
const priceMap = new Map<string, string>();
for (const row of latestPrices) {
  priceMap.set(row.supply_id, row.price);
}
```
Source: Existing pattern in `SuppliesService.findAll()` (lines 50-76) and `ProductsService.findAll()` (lines 76-100).

### Pattern 2: Module Composition (Avoid Circular Deps)
**What:** CostsModule imports ProductsModule and SuppliesModule to access their repos/services without circular dependencies.
**When to use:** CostsService needs BOM repo (from ProductsModule) and SupplyPriceHistory repo (from SuppliesModule).
**Key detail:** CostsModule should NOT have its own controller. Instead, either:
- Option A: CostsService is injected into ProductsController (via CostsModule importing into ProductsModule -- creates circular dep, avoid).
- Option B: CostsModule has its own CostsController with a `GET /costs/products` endpoint.
- Option C: ProductsController calls CostsService by having ProductsModule import CostsModule (CostsModule exports CostsService).

**Recommended:** Option C -- ProductsModule imports CostsModule. CostsModule imports TypeOrmModule.forFeature for SuppliesPerProductHistory and SupplyPriceHistory directly (not importing ProductsModule or SuppliesModule). This avoids circular deps entirely.

```typescript
// costs.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([SuppliesPerProductHistory, SupplyPriceHistory]),
  ],
  providers: [CostsService],
  exports: [CostsService],
})
export class CostsModule {}

// products.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([...existing...]),
    CostsModule, // <-- add
  ],
  ...
})
export class ProductsModule {}
```

### Pattern 3: Response DTO Extension
**What:** Extend the existing `ProductWithPrice` interface to include cost data.
**Example:**
```typescript
export interface ProductWithCost extends ProductWithPrice {
  cost: number | null;
  costBreakdown: CostBreakdownItem[] | null;
  costWarnings: string[];
}

export interface CostBreakdownItem {
  supplyId: string;
  supplyName: string;
  supplyType: string;
  quantity: number;
  unitType: string;
  unitPrice: number | null;
  lineCost: number | null;
  isSupplyActive: boolean;
}
```

### Pattern 4: Frontend Product Detail Page (Next.js App Router)
**What:** New route at `/productos/[id]/page.tsx` as a server component that fetches product data.
**Key:** Uses `apiFetch` for server-side data fetching (same as `/productos/page.tsx`).
```
nemea-front/src/app/(app)/productos/
├── page.tsx              # Product list (existing)
└── [id]/
    └── page.tsx          # Product detail (new)
```

### Anti-Patterns to Avoid
- **N+1 queries:** Never fetch BOM per product individually for the list view. The 2-query batch pattern must be used.
- **Cache layer:** No Redis, no in-memory cache -- on-the-fly is the locked decision.
- **Computed columns in DB:** No stored procedures, no materialized views -- pure application-level calculation.
- **WebSocket/SSE for live updates:** Navigation-based refresh is the locked decision.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Latest price per supply | Custom subquery join in TypeORM QueryBuilder | Raw SQL with DISTINCT ON via `.query()` | Pattern already proven in both SuppliesService and ProductsService. TypeORM QueryBuilder for DISTINCT ON is verbose and fragile. |
| Number formatting (ARS) | Custom format function | `toLocaleString('es-AR')` | Already used in `formatSellingPrice`. Extend pattern for costs. |
| Tooltip component | Custom hover tooltip | Shadcn/ui `<Tooltip>` | Already available in the project's component library |

## Common Pitfalls

### Pitfall 1: Decimal Arithmetic Precision
**What goes wrong:** JavaScript floating point math produces `0.1 + 0.2 = 0.30000000000000004`. Cost calculations (price x quantity) could show incorrect pennies.
**Why it happens:** TypeORM returns decimal columns as strings (confirmed in existing code -- `SupplyPriceHistory.price` is typed as `string`, `SuppliesPerProductHistory.quantity` is `string`). Converting to float and multiplying introduces rounding errors.
**How to avoid:** Parse to float, multiply, then round to 2 decimal places with `Math.round(value * 100) / 100`. This is sufficient for ~50-100 products with prices in ARS range. No need for a BigDecimal library at this scale.
**Warning signs:** Cost totals don't match manual calculation by 1 centavo.

### Pitfall 2: Circular Module Dependencies
**What goes wrong:** If CostsModule imports ProductsModule AND ProductsModule imports CostsModule, NestJS throws a circular dependency error.
**Why it happens:** CostsService needs BOM data (owned by ProductsModule) and ProductsController needs cost calculation (owned by CostsModule).
**How to avoid:** CostsModule should import TypeOrmModule.forFeature directly for the entities it needs (SuppliesPerProductHistory, SupplyPriceHistory), NOT import ProductsModule or SuppliesModule. ProductsModule then safely imports CostsModule.
**Warning signs:** `Nest cannot create the CostsModule instance. The module at index [X] is a circular dependency.`

### Pitfall 3: N+1 Query on Product List
**What goes wrong:** Fetching BOM per product in a loop for the list view generates N+1 queries.
**Why it happens:** Temptation to call `costsService.calculateForProduct(id)` in a map over products.
**How to avoid:** CostsService.calculateAll() must do 2 total queries: one for ALL active BOM entries, one for ALL latest supply prices. Then group and calculate in memory.
**Warning signs:** NestJS logger shows more than 2 SELECT statements for the product list endpoint.

### Pitfall 4: BOM Eager Loading Inconsistency
**What goes wrong:** The `SuppliesPerProductHistory` entity has `supply` as an eager relation but NOT `product`. When fetching all BOM entries, the product_id column must be accessed via the join column, not the relation.
**How to avoid:** Use `relations: ['product']` explicitly when fetching BOM entries for batch calculation, or select `product.id` in a custom query. The raw `.query()` approach avoids this entirely.
**Warning signs:** `product` is undefined on BOM entries.

### Pitfall 5: colSpan Mismatch After Adding Columns
**What goes wrong:** The ProductTypeGroup table currently has `const colSpan = 6`. Adding Costo and Margen columns makes it 8. The `ProductExpandedRow` receives `colSpan` as a prop -- if not updated, the expanded row won't span the full width.
**How to avoid:** Update `colSpan` constant in ProductTypeGroup AND pass the correct value to ProductExpandedRow.
**Warning signs:** Expanded row content doesn't align with table headers.

### Pitfall 6: Product Name as Link Breaking Click-to-Expand
**What goes wrong:** Currently, clicking a product row expands it. Making the product name a link to `/productos/:id` conflicts with the row click handler.
**How to avoid:** Use `e.stopPropagation()` on the link click, or make only the name cell a link (not the entire row). The row click continues to expand, but clicking the name specifically navigates to the detail page.
**Warning signs:** Clicking product name both navigates AND expands/collapses the row.

## Code Examples

### CostsService.calculateAll() - Core Method
```typescript
// Source: Composing existing patterns from SuppliesService and ProductsService
async calculateAll(): Promise<Map<string, ProductCostData>> {
  // Query 1: All active BOM entries with supply relation
  const bomEntries = await this.bomRepo.find({
    where: { isActive: true },
    relations: ['product', 'supply'],
  });

  // Query 2: Latest price per supply
  const latestPrices: { supply_id: string; price: string }[] =
    await this.priceHistoryRepo.query(
      `SELECT DISTINCT ON (supply_id) supply_id, price
       FROM supply_price_history
       ORDER BY supply_id, created_at DESC`,
    );

  const priceMap = new Map<string, number>();
  for (const row of latestPrices) {
    priceMap.set(row.supply_id, parseFloat(row.price));
  }

  // Group BOM by product
  const productBomMap = new Map<string, typeof bomEntries>();
  for (const entry of bomEntries) {
    const productId = entry.product.id;
    if (!productBomMap.has(productId)) {
      productBomMap.set(productId, []);
    }
    productBomMap.get(productId)!.push(entry);
  }

  // Calculate cost per product
  const result = new Map<string, ProductCostData>();
  for (const [productId, entries] of productBomMap) {
    const breakdown: CostBreakdownItem[] = [];
    let totalCost = 0;
    const warnings: string[] = [];

    for (const entry of entries) {
      const unitPrice = priceMap.get(entry.supply.id) ?? null;
      const quantity = parseFloat(entry.quantity);
      const lineCost = unitPrice !== null
        ? Math.round(unitPrice * quantity * 100) / 100
        : null;

      if (unitPrice === null) {
        warnings.push(`${entry.supply.name} sin precio`);
      }
      if (!entry.supply.isActive) {
        warnings.push(`${entry.supply.name} inactivo`);
      }

      if (lineCost !== null) {
        totalCost += lineCost;
      }

      breakdown.push({
        supplyId: entry.supply.id,
        supplyName: entry.supply.name,
        supplyType: entry.supply.type?.name ?? '',
        quantity,
        unitType: entry.supply.unitType,
        unitPrice,
        lineCost,
        isSupplyActive: entry.supply.isActive,
      });
    }

    result.set(productId, {
      cost: Math.round(totalCost * 100) / 100,
      costBreakdown: breakdown,
      costWarnings: warnings,
    });
  }

  return result;
}
```

### Frontend: formatCost and formatMargin Helpers
```typescript
// Extend types.ts with cost-related types and formatters
export interface ProductCost {
  cost: number | null;
  costBreakdown: CostBreakdownItem[] | null;
  costWarnings: string[];
}

export function formatCost(cost: number | null): string {
  if (cost === null) return '\u2014'; // em-dash
  return `$${cost.toLocaleString('es-AR', { minimumFractionDigits: 0, maximumFractionDigits: 2 })}`;
}

export function formatMargin(cost: number | null, price: number | null): { amount: string; percent: string } {
  if (cost === null || price === null) return { amount: '\u2014', percent: '\u2014' };
  const margin = price - cost;
  const percent = cost > 0 ? Math.round((margin / cost) * 100) : 0;
  return {
    amount: `$${margin.toLocaleString('es-AR', { minimumFractionDigits: 0, maximumFractionDigits: 2 })}`,
    percent: `${percent}%`,
  };
}
```

### Frontend: Product Detail Page Structure
```typescript
// app/(app)/productos/[id]/page.tsx
export default async function ProductDetailPage({
  params,
}: {
  params: Promise<{ id: string }>;
}): Promise<ReactElement> {
  const { id } = await params;
  const session = await auth();
  const isAdmin = session?.user?.role === 'admin';

  const [productRes, bomRes] = await Promise.all([
    apiFetch<{ data: ProductWithCost }>(`/api/products/${id}`),
    apiFetch<{ data: BomItem[] }>(`/api/products/${id}/bom`),
  ]);

  // Render header + BOM table with cost columns + margin summary
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| TypeORM subquery joins for latest price | Raw SQL DISTINCT ON via `.query()` | Phase 4 (04-01) | Simpler, fewer queries, PostgreSQL-native |
| Per-product BOM fetch (N+1) | Batch BOM fetch for all products | This phase | Must follow established batch pattern |
| Selling price only in list | Cost + margin columns in list | This phase | Column count increases from 6 to 8 |

## Open Questions

1. **GET /products response shape: flat vs nested cost data?**
   - What we know: Current `ProductWithPrice` extends Product with `currentPrice` and `lastPriceUpdate`. Cost data needs to be added similarly.
   - What's unclear: Should `costBreakdown` be included in the list response (it's an array per product, could be large) or only in the detail/BOM endpoint?
   - Recommendation: Include `cost` (number) and `costWarnings` (string[]) in list response. Exclude `costBreakdown` from list -- it's already available via `GET /products/:id/bom` and would bloat the list response. The expanded row already fetches BOM lazily. Enrich the BOM response with price data instead.

2. **Product detail page data source: separate endpoint or reuse?**
   - What we know: `GET /products/:id` (findOne) currently returns a bare Product without price or cost data.
   - What's unclear: Should we enrich findOne to include cost + price, or use separate calls?
   - Recommendation: Enrich findOne to return `ProductWithCost` (consistent with findAll pattern), detail page also fetches BOM separately for breakdown.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30 + ts-jest |
| Config file | `nemea-back/package.json` (jest section) |
| Quick run command | `cd nemea-back && npx jest --testPathPattern=costs --no-coverage` |
| Full suite command | `cd nemea-back && npx jest --no-coverage` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| COST-01 | CostsService.calculateAll returns correct cost from BOM x latest prices | unit | `npx jest --testPathPattern=costs.service.spec --no-coverage` | No -- Wave 0 |
| COST-01 | Handles product without BOM (returns null cost) | unit | Same as above | No -- Wave 0 |
| COST-01 | Handles supply without price (partial cost + warning) | unit | Same as above | No -- Wave 0 |
| COST-02 | GET /products includes cost and costWarnings fields | unit | `npx jest --testPathPattern=products.controller.spec --no-coverage` | No -- Wave 0 |
| COST-03 | Cost recalculates on each request (no cache) | unit | Covered by COST-01 test with different price maps | No -- Wave 0 |
| COST-04 | GET /products/:id returns cost breakdown | unit | Same as COST-02 | No -- Wave 0 |

### Sampling Rate
- **Per task commit:** `cd nemea-back && npx jest --testPathPattern=costs --no-coverage`
- **Per wave merge:** `cd nemea-back && npx jest --no-coverage`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `nemea-back/src/costs/costs.service.spec.ts` -- covers COST-01, COST-03
- [ ] `nemea-back/src/costs/costs.module.spec.ts` -- module composition verification (optional)
- [ ] Update `nemea-back/src/products/products.controller.spec.ts` if it exists -- would need updating for cost fields, but no existing spec for products controller

## Sources

### Primary (HIGH confidence)
- Existing codebase: `nemea-back/src/supplies/supplies.service.ts` -- DISTINCT ON batch pattern (lines 50-76)
- Existing codebase: `nemea-back/src/products/products.service.ts` -- ProductWithPrice pattern, findAll batch query (lines 64-101)
- Existing codebase: `nemea-back/src/products/entities/supplies-per-product-history.entity.ts` -- BOM entity with eager supply relation
- Existing codebase: `nemea-front/src/components/products/ProductTypeGroup.tsx` -- Current table structure (6 columns, colSpan=6)
- Existing codebase: `nemea-front/src/components/products/ProductExpandedRow.tsx` -- BOM display, lazy fetch pattern

### Secondary (MEDIUM confidence)
- NestJS module composition patterns (TypeOrmModule.forFeature for cross-module entity access without circular deps)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- zero new dependencies, all patterns exist in codebase
- Architecture: HIGH -- composing proven patterns (DISTINCT ON, batch fetch, module separation)
- Pitfalls: HIGH -- identified from actual codebase analysis (colSpan, circular deps, N+1, decimal math)
- Frontend: HIGH -- extending existing components with new columns and a standard detail page

**Research date:** 2026-03-06
**Valid until:** 2026-04-06 (stable -- no external dependencies to go stale)
