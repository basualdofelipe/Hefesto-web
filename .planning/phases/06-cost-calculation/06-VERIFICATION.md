---
phase: 06-cost-calculation
verified: 2026-03-06T22:00:00Z
status: passed
score: 4/4 success criteria verified
re_verification: false
---

# Phase 6: Cost Calculation Verification Report

**Phase Goal:** Every product displays its current real cost calculated dynamically from the latest supply prices -- the core value of the app
**Verified:** 2026-03-06T22:00:00Z
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths (Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | The product list shows a calculated cost column for every product, computed from current supply prices | VERIFIED | `ProductsController.findAll()` calls `costsService.calculateAll()` and merges cost data into each product. `ProductTypeGroup.tsx` renders 8 columns including Costo (line 152) and Margen (line 154). `formatCost()` formats cost from the `product.cost` field. |
| 2 | After updating a supply price, the product list immediately reflects the new cost -- no manual action required | VERIFIED | Cost is calculated on-the-fly in `CostsService.calculateAll()` via `DISTINCT ON (supply_id) ... ORDER BY supply_id, created_at DESC` (line 34 of costs.service.ts). No caching layer. The productos page is a Next.js server component fetching fresh data on each navigation. |
| 3 | The product detail page shows a cost breakdown listing each supply, its quantity, its current price, and its line cost | VERIFIED | `ProductsController.findOne()` returns full `costBreakdown` array (not null like list). `ProductDetailClient.tsx` renders BOM table with columns: Insumo, Tipo, Cantidad, Unidad, Precio Unit., Costo Linea (lines 171-178). Total row at bottom (line 227-236). Margin summary card with Costo total, Precio de venta, Margen (lines 251-285). |
| 4 | NestJS logs confirm cost calculation uses exactly 2 SQL queries for the entire product list (no N+1 queries) | VERIFIED | `CostsService.calculateAll()` executes exactly 2 queries: Query 1 = `bomRepo.find()` for all active BOM entries (line 22-25), Query 2 = raw SQL `DISTINCT ON` for latest prices (line 33-37). All further computation is in-memory via `buildCostMap()`. `calculateForProduct()` also uses 2 queries with filtered `WHERE supply_id IN (...)`. |

**Score:** 4/4 success criteria verified

### Required Artifacts (Plan 06-01)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/costs/costs.module.ts` | CostsModule with TypeOrmModule.forFeature | VERIFIED | 14 lines. Imports SuppliesPerProductHistory + SupplyPriceHistory repos. Exports CostsService. |
| `nemea-back/src/costs/costs.service.ts` | 2-query batch cost calculation | VERIFIED | 141 lines. calculateAll(), calculateForProduct(), buildCostMap(). DISTINCT ON raw SQL. Proper decimal rounding. |
| `nemea-back/src/costs/dto/product-with-cost.dto.ts` | ProductWithCost and CostBreakdownItem interfaces | VERIFIED | 24 lines. CostBreakdownItem, ProductCostData, ProductWithCost interfaces defined. |
| `nemea-back/src/costs/costs.service.spec.ts` | Unit tests | VERIFIED | 248 lines. 8 test cases covering: 2-BOM-item cost, no BOM, missing price warning, inactive supply warning, decimal rounding, single product calc, null for missing. |
| `nemea-back/src/app.module.ts` | CostsModule registered | VERIFIED | CostsModule imported at line 8, added to imports array at line 34 (before ProductsModule). |
| `nemea-back/src/products/products.module.ts` | CostsModule imported for DI | VERIFIED | CostsModule imported at line 12, added to module imports at line 18. |
| `nemea-back/src/products/products.controller.ts` | Enriched findAll/findOne with cost data | VERIFIED | CostsService injected. findAll() returns ProductWithCost[] (line 59). findOne() returns ProductWithCost with full costBreakdown (line 171). |
| `nemea-back/src/products/products.service.ts` | findOneWithPrice method | VERIFIED | findOneWithPrice() at line 115-132. Fetches product + latest price via raw SQL. |

### Required Artifacts (Plan 06-02)

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-front/src/components/products/types.ts` | ProductWithCost type, formatCost, formatMargin helpers | VERIFIED | Product interface extended with cost, costBreakdown, costWarnings. CostBreakdownItem interface. formatCost() and formatMargin() helper functions. |
| `nemea-front/src/components/products/ProductTypeGroup.tsx` | Cost/Margin columns, group header avg cost | VERIFIED | 8-column layout (colSpan=8). Costo and Margen columns. avgCost computed via useMemo. "Costo prom" badge in group header. Tooltip for products without BOM. AlertTriangle for warnings. Product name as Link. |
| `nemea-front/src/components/products/ProductExpandedRow.tsx` | Enriched BOM table with Precio Unit. and Costo Linea | VERIFIED | Parallel fetch of BOM + product detail for costBreakdown. 6-column BOM table (Insumo, Tipo, Cantidad, Unidad, Precio Unit., Costo Linea). Total row. Margin summary below table. |
| `nemea-front/src/app/(app)/productos/[id]/page.tsx` | Product detail page server component | VERIFIED | Fetches product, BOM, catalogs, supplies via Promise.all. Passes to ProductDetailClient. Handles params as Promise (Next.js 16 pattern). |
| `nemea-front/src/components/products/ProductDetailClient.tsx` | Detail page client component | VERIFIED | 397 lines. Header with product name + SKU + status badge. BOM cost breakdown table. Margin summary card. Admin actions (edit, BOM, price, history, toggle status). Back link to /productos. Cost warnings display. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `costs.service.ts` | `supply_price_history` table | DISTINCT ON raw SQL | WIRED | Line 34: `SELECT DISTINCT ON (supply_id) supply_id, price FROM supply_price_history ORDER BY supply_id, created_at DESC` |
| `costs.service.ts` | `supplies_per_product_history` table | TypeORM find | WIRED | Line 22: `bomRepo.find({ where: { isActive: true }, relations: ['product', 'supply'] })` |
| `products.controller.ts` | `costs.service.ts` | DI injection | WIRED | Line 41: `private readonly costsService: CostsService`. Line 63: `costsService.calculateAll()`. Line 173: `costsService.calculateForProduct(id)`. |
| `productos/page.tsx` (list) | `/api/products` | apiFetch | WIRED | Line 28: `apiFetch<{ data: Product[] }>('/api/products?includeInactive=true')` |
| `productos/[id]/page.tsx` | `/api/products/:id` | apiFetch | WIRED | Line 44: `apiFetch<{ data: ProductWithCostResponse }>(\`/api/products/${id}\`)` |
| `ProductTypeGroup.tsx` | `types.ts` | formatCost, formatMargin | WIRED | Lines 35: imports formatCost, formatMargin, formatSellingPrice from './types' |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| COST-01 | 06-01 | System calculates product cost dynamically: SUM(latest_price(supply) x quantity) for all active materials | SATISFIED | CostsService.calculateAll() fetches all BOM entries + latest prices via DISTINCT ON, computes lineCost = unitPrice * quantity for each, sums to total cost. |
| COST-02 | 06-01, 06-02 | Product list shows calculated cost per product | SATISFIED | ProductsController.findAll() merges cost data. ProductTypeGroup renders Costo column. |
| COST-03 | 06-01, 06-02 | When a supply price is updated, all products using that supply reflect the new cost automatically | SATISFIED | Cost is calculated on-the-fly from latest prices (DISTINCT ON ... ORDER BY created_at DESC). No cache. Server component fetches fresh on navigation. |
| COST-04 | 06-02 | Product detail shows cost breakdown by material | SATISFIED | ProductsController.findOne() returns full costBreakdown. ProductDetailClient renders BOM table with Precio Unit. and Costo Linea columns, total row, and margin summary card. |

No orphaned requirements found. REQUIREMENTS.md maps COST-01 through COST-04 to Phase 6, and all four are covered by plans 06-01 and 06-02.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None | - | - | - | No anti-patterns detected |

All `return null` instances are legitimate business logic (null cost for products without BOM, null average for groups without costed products). No TODOs, FIXMEs, placeholders, or stub implementations found.

### Human Verification Required

### 1. Visual verification of cost columns in product list

**Test:** Navigate to /productos. Verify 8 columns: SKU, Nombre, Terminacion, Color, Talle, Costo, Precio Venta, Margen.
**Expected:** Products with BOM show formatted cost and margin ($ amount + %). Products without BOM show em-dash with "Sin materiales definidos" tooltip. Group headers show "Costo prom: $X.XXX" badge.
**Why human:** Visual layout, column alignment, and tooltip behavior cannot be verified programmatically.

### 2. Cost auto-refresh after supply price update

**Test:** Note a product's cost. Go to /insumos, update a supply price used by that product. Navigate back to /productos.
**Expected:** The product's cost reflects the new supply price immediately.
**Why human:** Requires running backend + frontend and performing a multi-page flow.

### 3. Product detail page at /productos/:id

**Test:** Click a product name in the list. Verify detail page loads with header, BOM cost breakdown table, margin summary card, and admin actions.
**Expected:** BOM table shows Insumo, Tipo, Cantidad, Unidad, Precio Unit., Costo Linea with total row. Margin card shows Costo total, Precio de venta, Margen. "Volver a productos" link works.
**Why human:** Visual layout and navigation behavior require manual testing.

### 4. Expanded row BOM enrichment

**Test:** Click a product row to expand it. Verify BOM table shows cost columns with margin summary below.
**Expected:** Enriched BOM with Precio Unit. and Costo Linea columns. Total row. Margin summary (Costo total, Precio venta, Margen).
**Why human:** Lazy-loaded data and visual rendering require browser interaction.

### Gaps Summary

No gaps found. All success criteria are verified through code inspection:

1. **Backend cost engine:** CostsService implements the exact 2-query batch pattern (BOM find + DISTINCT ON prices) with in-memory grouping and cost calculation. Proper decimal rounding. Handles missing prices and inactive supplies with warnings.

2. **API enrichment:** ProductsController.findAll() and findOne() merge cost data from CostsService. List returns cost + costWarnings (costBreakdown null for performance). Detail returns full costBreakdown.

3. **Frontend display:** Product list has 8 columns with Costo and Margen. Expanded rows fetch costBreakdown in parallel. Product detail page provides full breakdown with margin summary. Group headers show average cost.

4. **Auto-refresh:** No caching layer anywhere -- cost is calculated on-the-fly from current prices. Server component fetches fresh on each navigation.

5. **Tests:** 8 unit tests cover all cost scenarios (2-item product, no BOM, missing prices, inactive supplies, decimal rounding, single product).

All 5 commits verified in sub-repos (3 backend, 2 frontend). All requirement IDs (COST-01 through COST-04) accounted for and satisfied.

---

_Verified: 2026-03-06T22:00:00Z_
_Verifier: Claude (gsd-verifier)_
