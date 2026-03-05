# Phase 4: Supplies and Price History - Context

**Gathered:** 2026-03-05
**Status:** Ready for planning

<domain>
## Phase Boundary

CRUD for supplies (with type FK, supplier FK, notes, unit_type, is_active) and append-only price history. Every price change creates a new immutable record; current price = most recent. Frontend shows supplies in a grouped expandable table with inline actions.

</domain>

<decisions>
## Implementation Decisions

### Supply list layout
- Grouped table by supply type (Cuero, Herraje, Packaging, etc.) with collapsible sections
- Each section header shows type name and count (e.g., "Cuero (3 insumos)")
- Table columns: Nombre, Proveedor, Precio Actual, Estado
- Click on a row expands it inline (no page navigation)

### Expand inline content
- Notas (text field from supply)
- Proveedor name as clickeable link (navigates to /proveedores/:id/editar)
- Precio actual with unit (e.g., $15.200/m²)
- Última actualización date
- Action buttons: [Editar] [+ Nuevo precio] [Ver historial] [Estado toggle]

### Supply data model
- `notes`: text field, nullable — free-form notes for all supply types (e.g., "2mm espesor, flor natural")
- `unit_type`: enum on the supply entity (m², unidad, metro, kg) — fixed per supply, not per price record
- `is_active`: boolean with partial unique index on (name, supplier_id) WHERE is_active = true
- Unique constraint includes supplier_id — allows same supply name with different suppliers

### Filtering & organization
- Search box by supply name
- Dropdown filter by supplier
- Toggle "Mostrar inactivos" — off by default (only active supplies shown)
- Inactive supplies appear dimmed when toggle is on

### Create/edit supply
- Both use a modal/dialog over the list (not a separate page)
- Create fields: nombre (required), tipo (select), proveedor (select), unidad (select), notas (textarea), precio inicial (optional)
- Edit: same modal, prefilled with existing data. Editable fields: nombre, tipo, unidad, notas
- Proveedor is NOT editable after creation — if supplier changes, deactivate old supply and create new one

### Supply-supplier relationship
- Supplier is fixed once a supply is created (not editable)
- Changing supplier = deactivate old supply + create new supply with new supplier
- This preserves price history per supplier relationship
- Unique index on (name, supplier_id) WHERE is_active = true allows "Cuero Vacheta" with Curtiembre X and "Cuero Vacheta" with Curtiembre Y

### Price update flow
- "+" Nuevo precio" button in expand inline opens an input field with confirm/cancel
- Admin enters new price amount — creates a new record in price history
- Price is optional at supply creation (supply can exist without price)
- Price history is append-only — no editing or deleting past prices

### Price history display
- Accessed via "Ver historial" button in expand → opens a modal
- Simple list: date + amount, sorted newest to oldest
- No visual indicators (arrows, colors, percentages) — just dates and amounts
- Unit shown from the supply's unit_type

### Soft delete behavior
- Toggle switch in the expanded row (same pattern as suppliers)
- Toast confirmation on toggle
- Block deactivation if supply is used in active product BOM (check implemented in Phase 5, free toggle in Phase 4)
- Deactivating a supplier cascades: all its active supplies are also deactivated
- Reactivating a supplier does NOT reactivate its supplies — admin reactivates supplies individually

### Sidebar navigation
- "Insumos" added to sidebar under a "Datos base" group header
- Group contains: Catálogos, Proveedores, Insumos
- Future phases add items below (Productos in Phase 5)

### Claude's Discretion
- Exact styling of grouped table headers and collapsible animations
- Modal size and layout for create/edit and price history
- Loading and error states
- Toast messages and duration
- Search debounce timing
- Collapsible section default state (all open vs all closed)
- Empty state design for supplies list

</decisions>

<specifics>
## Specific Ideas

- The expand inline pattern is chosen over separate detail pages — keeps the user in context of the list
- Supplier link in expand navigates to the existing supplier edit page
- Supply-supplier coupling is intentional: price history always belongs to one supplier relationship
- "Datos base" sidebar group organizes reference data sections together

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BaseEntity` (id UUID, created_at, updated_at): Supply and SupplyPriceHistory extend this
- `Supplier` entity: FK target, already has is_active + partial unique index pattern
- `SupplyType` entity: FK target, already exists in catalogs module
- `@Roles(Role.ADMIN)` decorator: Write endpoints restricted to ADMIN
- Shadcn/ui: Dialog, Table, Input, Button, Select, Collapsible, Tabs components
- `apiClientFetch` wrapper: For mutations from client components
- `react-hook-form` + `zod`: For supply create/edit modal validation
- `sonner` Toaster: Toast notifications already wired

### Established Patterns
- Backend: NestJS module (controller + service + entity + DTO + module) registered in AppModule
- Backend: Partial unique index with is_active for soft delete (suppliers pattern)
- Backend: Swagger decorators on all controllers
- Backend: Error messages in Spanish
- Frontend: Server-component data fetching + client-component interactivity
- Frontend: Inline CRUD pattern (catalogs) and separate page pattern (suppliers) — this phase uses expand inline + modal (new pattern)
- Frontend: Route groups (app)/(auth) for sidebar layout

### Integration Points
- `AppModule.imports`: Register new SuppliesModule
- `SuppliersModule`: Need to export SuppliersService (or add cascade logic) for deactivation cascade
- Sidebar component: Add "Datos base" group header + "Insumos" item
- Frontend routing: New page at `/insumos`

</code_context>

<deferred>
## Deferred Ideas

- Structured supply specs by type (terminación, espesor, colores for leather; dimensions for packaging) — v2 feature, requires conditional fields per supply type and new catalog entities
- Visual price trend indicators (arrows, percentages, sparklines) in history — v2 polish

</deferred>

---

*Phase: 04-supplies-and-price-history*
*Context gathered: 2026-03-05*