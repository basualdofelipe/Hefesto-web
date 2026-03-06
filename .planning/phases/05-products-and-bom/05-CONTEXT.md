# Phase 5: Products and BOM - Context

**Gathered:** 2026-03-05
**Status:** Ready for planning

<domain>
## Phase Boundary

Product CRUD with 5 catalog dimension FKs (type, name, finish, color, size), auto-generated numeric SKU, material composition (BOM) with version history, and selling price with history. Batch product creation (multiple colors/sizes at once). BOM group editing. No cost calculation yet (Phase 6).

</domain>

<decisions>
## Implementation Decisions

### SKU generation
- Format: `{type.sku_code}.{name.sku_code}.{finish.sku_code}.{color.sku_code}.{size.sku_code}` (e.g., `1.2.1.3.0`)
- Each catalog entity gets a new `sku_code` field (smallint, unique) assigned manually by admin when creating catalog items
- Migration adds `sku_code` column to all 5 product dimension tables + product_size
- SKU auto-generated on product creation by concatenating the sku_codes of selected dimensions
- "Talle Unico" seeded as default size with sku_code 0
- Dimensions are editable after creation — SKU regenerates automatically on edit
- `sku_code` stored as VARCHAR in product table, unique constraint

### Product list layout
- Grouped table by product type (same pattern as supplies) with collapsible sections
- Section header shows type name and count (e.g., "Billeteras (4 productos)")
- Table columns: SKU, Nombre, Terminacion, Color, Talle, Precio Venta
- Precio column shows "—" if no price loaded yet
- Search by product name + toggle "Mostrar inactivos" (same filter pattern as supplies)

### Product expand inline
- Click row expands showing: BOM table (insumo, cantidad, unidad), precio de venta actual, action buttons
- Actions: [Editar] [Editar BOM] [+ Precio venta] [Historial precios] [Estado toggle]
- Inactive supplies in BOM shown dimmed with warning indicator
- Supplies without price shown as "sin precio" in BOM lines

### Product creation (batch)
- Modal dialog over the list
- Step 1: Select tipo + nombre + terminacion (fixed for the batch)
- Step 2: Checkbox all desired colores
- Step 3: Checkbox all desired talles
- Preview shows all SKUs that will be generated with product names
- "Crear N productos" button creates all combinations at once
- BOM is optional at creation time — can be added later
- All products in a batch receive a copy of the same BOM (if one is set by existing products or template)

### Product editing
- Same modal as creation but for single product
- All 5 dimensions editable — SKU regenerates on save
- Unique constraint on sku_code prevents duplicate combinations

### BOM editor
- Modal with editable table opened via "Editar BOM" button in expand inline
- Rows: select insumo (Combobox with search, grouped by supply type) + cantidad input + unidad (auto-filled from supply.unitType, read-only)
- "+ Agregar material" button adds a new row
- Same supply can appear multiple times (no unique constraint on product_id + supply_id) — useful when same leather used in different pieces
- Save creates new BOM version silently: old records marked is_active=false, new records created is_active=true (transaction)
- BOM history stored in DB but NOT shown in UI for now — deferred

### BOM group editing
- Button "Editar BOM grupal" in the group header (e.g., Billeteras section)
- Opens modal with checkboxes of all products in the group, all checked by default
- Products with BOM different from the majority get a warning indicator
- Admin unchecks products to exclude from the change
- Standard BOM editor below the selection
- Save applies the new BOM to all checked products

### Supply deactivation guard (enhancement to Phase 4)
- Block deactivation of a supply that is in any active BOM
- Error message: "Este insumo esta en uso por X productos"
- Admin must remove supply from all BOMs first, then deactivate

### Selling price
- Inline "+" button in expand opens input field with confirm/cancel (same pattern as supply price)
- Append-only: new price creates a new record in product_price_history, previous preserved
- Current price = most recent record
- Price history modal: same component pattern as PriceHistoryDialog (date + amount, newest first)
- Batch price update: button in group header, checkboxes to select products, one amount applied to all selected
- Currency field in DB (`currency` VARCHAR default 'ARS') but NOT shown in UI — prepared for future multi-currency

### Navigation
- Single page at `/productos` — no subroutes
- Everything via expand inline + modals
- "Productos" added to sidebar under "Datos base" group (below Insumos): Catalogos, Proveedores, Insumos, Productos

### Seed data
- Seed real products from Costos Nemea.xlsx with their materials (BOM)
- Seed sku_code values for existing catalog items
- Seed runs as TypeORM migration (idempotent, ON CONFLICT DO NOTHING)

### Claude's Discretion
- Exact styling of group headers, expand rows, and modals
- Loading and error states
- Toast messages and duration
- Empty state design for products list and BOM
- Search debounce timing
- Combobox design for supply selection in BOM editor
- How to detect "majority BOM" for group edit warning
- Collapsible sections default state

</decisions>

<specifics>
## Specific Ideas

- SKU format `1.2.1.3.0` comes from user's existing Google Sheets system — admin assigns numeric codes to each catalog item
- Batch creation is essential: user doesn't want to create one product per color/size combination manually
- BOM group editing with checkboxes + warning for different BOMs is key — same leather change usually applies to all colors of a product
- The pattern "copias independientes del BOM por producto" allows individual overrides while keeping batch edits simple
- Supply deactivation guard was planned since Phase 4 (toggle was left free, guard added in Phase 5)

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BaseEntity` (id UUID, created_at, updated_at): Product, SuppliesPerProductHistory, ProductPriceHistory extend this
- `Supply` entity with `unitType` enum (m2, unidad, metro, kg): BOM inherits unit from supply
- `SupplyPriceHistory` entity: Pattern for ProductPriceHistory (append-only, decimal price)
- `SupplyTypeGroup` component: Reusable grouped collapsible table pattern for product types
- `SupplyExpandedRow` component: Pattern for ProductExpandedRow
- `SupplyFormDialog` component: Pattern for ProductFormDialog (modal CRUD)
- `AddPriceInline` component: Reusable for product selling price addition
- `PriceHistoryDialog` component: Reusable for product price history modal
- `types.ts` shared types pattern: Same approach for product types
- 5 catalog entities (ProductType, ProductName, ProductFinish, ProductColor, ProductSize): FK targets, need `sku_code` field added
- `apiClientFetch` wrapper: For mutations
- `react-hook-form` + `zod`: For product create/edit and BOM editor validation

### Established Patterns
- Backend: NestJS module (controller + service + entity + DTO + module) registered in AppModule
- Backend: Partial unique index with is_active for soft delete
- Backend: Swagger decorators on all controllers
- Backend: Error messages in Spanish
- Backend: 2-query pattern (find + DISTINCT ON) for current price
- Frontend: Server-component data fetching + client-component interactivity
- Frontend: Grouped expandable table with collapsible sections (supplies pattern)
- Frontend: Modal for create/edit (supplies pattern)
- Frontend: Inline price addition + price history modal (supplies pattern)
- Frontend: Route groups (app)/(auth) for sidebar layout

### Integration Points
- `AppModule.imports`: Register new ProductsModule
- `SuppliesModule`: Need to add BOM check before supply deactivation (guard)
- `CatalogsModule`: Migration adds `sku_code` field to all 5 dimension entities + update DTOs
- Sidebar component: Add "Productos" item under "Datos base" group
- Frontend routing: New page at `/productos`

</code_context>

<deferred>
## Deferred Ideas

- BOM version history UI — records stored in DB (is_active=false) but no UI to view old versions yet
- Margin calculation (precio venta - costo) — Phase 6 when cost is calculated
- Price per channel (Tiendanube, B2B, direct) — v2 when B2B clients and Tiendanube are implemented
- Visual price trend indicators (arrows, percentages) — v2 polish

</deferred>

---

*Phase: 05-products-and-bom*
*Context gathered: 2026-03-05*
