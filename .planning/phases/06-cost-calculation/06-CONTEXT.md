# Phase 6: Cost Calculation - Context

**Gathered:** 2026-03-06
**Status:** Ready for planning

<domain>
## Phase Boundary

Dynamic cost per product calculated from latest supply prices (the core value of the app). Includes cost column in product list, cost breakdown in expanded row and new product detail page, margin calculation (bruto, sin IVA), and group header cost aggregation. IVA, comisiones Tiendanube, IIBB, and credit fiscal belong in Phase 7.

</domain>

<decisions>
## Implementation Decisions

### Cost & margin columns in product list
- Add Costo column next to Precio Venta in the product list table
- Add Margen column showing $ and % (markup sobre costo: margen/costo x 100)
- Column order: SKU | Nombre | Terminacion | Color | Talle | Costo | Precio Venta | Margen
- All values sin IVA — prices are stored without IVA, cost is raw materials sum
- Margin = Precio Venta - Costo. Margen % = (Margen / Costo) x 100
- IVA, comisiones, and net profit calculation deferred to Phase 7 (calculadora Tiendanube)

### Cost breakdown in expanded row
- Enrich existing BOM table with two new columns: Precio Unitario and Costo Linea
- Each BOM row shows: Insumo | Cantidad | Unidad | Precio Unit. | Costo Linea
- Total row at the bottom summing all line costs
- Margin summary below: Costo total, Precio Venta, Margen $ y %

### Product detail page
- New page at `/productos/:id` — navigated to by clicking product name in the list (name becomes link)
- Layout: single column, stacked sections
- Sections: Product header (full name + SKU + status) -> BOM table with cost breakdown -> Margin summary
- Back navigation: "Volver a productos" link at top
- All admin actions available: edit product, edit BOM, add selling price, toggle status
- Reuses same modal components from the list page (ProductFormDialog, BomEditorDialog, etc.)

### Cost calculation method
- On-the-fly calculation per request — no cache, no background job
- 2 SQL queries + in-memory calculation: 1) active BOM entries for all products, 2) DISTINCT ON latest price per supply
- Always fresh — no stale data, no cache invalidation needed
- ~50-100 products with ~20 supplies = fast enough (<50ms)
- CostsModule separate from ProductsModule (imports both SuppliesModule + ProductsModule, avoids circular deps)

### Cost update behavior
- Auto-refresh on navigation — product list fetches fresh data on page load
- No WebSocket, no polling, no manual refresh button
- Update supply price on /insumos -> navigate to /productos -> costs already reflect new price

### Group header aggregation
- Product type group headers show count + average cost
- Format: "Billeteras (4 productos) — Costo prom: $4.800"
- Average excludes products without BOM (don't drag the average down with $0)

### Edge cases
- Product without BOM: show "—" in cost and margin columns, tooltip "Sin materiales definidos"
- Supply without price in BOM: show partial cost with warning indicator, tooltip lists which supplies lack price
- Inactive supply in BOM: included in cost calculation (has last known price), shown dimmed with warning in breakdown
- Product without selling price: show "—" in precio and margin columns (existing behavior)

### Claude's Discretion
- Exact styling of cost/margin columns (color coding, font weight)
- Warning indicator design (icon choice, tooltip implementation)
- Product detail page exact spacing and card styling
- Loading states for cost calculation
- How average cost is displayed in group headers (font size, color)
- BOM table styling in product detail page vs expand inline (may differ slightly)

</decisions>

<specifics>
## Specific Ideas

- Supply prices are loaded sin IVA (proveedores facturan en blanco, admin carga precio sin IVA)
- Selling prices are also sin IVA (precio que se pone en Tiendanube, no incluye IVA)
- Margin is "bruto" — just materials cost vs selling price, no taxes/comissions
- Phase 7 will add calculadora Tiendanube with ganancia min/max range by payment method (debito, credito, cuotas sin interes, transferencia) and shipping cost variations
- The cost from Phase 6 is the base input that Phase 7's calculator needs

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProductsService.findAll()`: Already does DISTINCT ON batch query for selling prices — same pattern for supply prices
- `SuppliesPerProductHistory` entity: BOM with eager `supply` relation, `quantity` as decimal, `isActive` flag
- `Supply` entity: Has `unitType` enum, `isActive`, and relation to `SupplyPriceHistory`
- `SupplyPriceHistory`: DISTINCT ON pattern for latest price already implemented in `SuppliesService`
- Frontend: `ProductTable`, `ProductTypeGroup`, `ProductExpandedRow` — all need cost columns added
- Frontend: `BomEditorDialog`, `AddPriceInline`, `PriceHistoryDialog` — reusable on detail page
- `apiClientFetch` and `apiFetch` wrappers for client/server fetching

### Established Patterns
- Backend: 2-query pattern (find + DISTINCT ON) for latest prices — apply to supply prices for cost calc
- Backend: NestJS module separation (CostsModule imports ProductsModule + SuppliesModule)
- Backend: Swagger decorators, Spanish error messages, @Roles(Role.ADMIN) for write endpoints
- Frontend: Server-component data fetching + client-component interactivity
- Frontend: Grouped expandable table with collapsible sections
- Frontend: Modal for CRUD operations (same modals reused on detail page)

### Integration Points
- `AppModule.imports`: Register new CostsModule
- `ProductsController`: Modify GET /products to include cost data (or new endpoint via CostsController)
- `ProductsService`: Already exported for external module consumption
- Frontend `/productos/page.tsx`: Add cost/margin columns to ProductTable
- Frontend: New route `/productos/[id]/page.tsx` for product detail page
- Sidebar: No changes needed (Productos link already exists)

</code_context>

<deferred>
## Deferred Ideas

- Calculadora Tiendanube with ganancia min/max range by payment method and shipping — Phase 7
- IVA on supply costs (credito fiscal) and IVA on selling price — Phase 7
- IIBB deduction on sales — Phase 7
- Comisiones MercadoPago/Tiendanube by payment method — Phase 7
- BOM version history UI (old compositions) — future phase
- Cost trend over time (how product cost evolved as supply prices changed) — v2
- Export cost data to Excel/PDF — v2

</deferred>

---

*Phase: 06-cost-calculation*
*Context gathered: 2026-03-06*
