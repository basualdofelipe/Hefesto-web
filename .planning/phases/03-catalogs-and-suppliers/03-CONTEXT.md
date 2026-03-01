# Phase 3: Catalogs and Suppliers - Context

**Gathered:** 2026-03-01
**Status:** Ready for planning

<domain>
## Phase Boundary

CRUD for all 5 product dimensions (product_type, product_name, product_finish, product_color, product_size), supply types (supply_type), and suppliers. All reference data must exist and be manageable before any supply or product is created in later phases.

</domain>

<decisions>
## Implementation Decisions

### Catalog UI layout
- Single page at `/catalogos` with 6 tabs: Tipos, Nombres, Terminaciones, Colores, Talles, Tipos de Insumo
- All 6 catalogs are simple name-only tables presented identically within their tab
- Reuse existing Shadcn/ui Tabs component

### Catalog editing
- Inline editing: click the item name to edit in-place, shows input + confirm/cancel
- "Agregar" button shows an inline input at the top of the list for new items
- No modal or separate form needed — each catalog item is just a single name field

### Catalog deletion
- Block deletion when the item is referenced by a product or supply
- Show explanatory message: "No se puede eliminar: usado por X productos"
- Admin must remove the FK reference first before deleting the catalog entry

### Catalog read-only access
- USER role can see the catalog page with all tabs and lists
- Edit, delete, and add buttons are hidden for USER role (backend also enforces via @Roles)

### Supplier list
- Table format with columns: Nombre, Email, Telefono, WhatsApp, Estado
- Simple search box at the top that filters the table as you type by supplier name
- Inactive suppliers remain visible but dimmed in the table

### Supplier form
- Separate page: `/proveedores/nuevo` for create, `/proveedores/:id/editar` for edit
- Full form with all fields: nombre (required), direccion, email, telefono, whatsapp, descripcion
- Back button returns to supplier list
- Cancelar/Guardar buttons at the bottom

### Supplier deactivation
- Toggle switch or status badge directly in the table row
- Click to toggle active/inactive with confirmation toast
- Inactive suppliers stay visible in the list but appear dimmed
- Can reactivate the same way

### Navigation
- Collapsible sidebar on the left with icons + labels
- Logo moves to sidebar top
- Toggle button to collapse to icons-only mode
- Only show sections that are built (no disabled future items)
- Phase 3 sidebar items: Inicio, Catalogos, Proveedores
- Future phases add their items when built
- Mobile: hamburger icon in header opens sidebar as overlay/drawer

### Seed data
- Seed all 6 catalog dimensions with real business data extracted from `Costos Nemea.xlsx`
- Also seed supplier data from the Excel
- Idempotent seed: use INSERT ... ON CONFLICT DO NOTHING (safe to re-run)
- Seed runs as a TypeORM migration

### Claude's Discretion
- Exact Tailwind styling, spacing, and color palette for active/inactive states
- Loading and error states
- Toast notification messages and duration
- Sidebar animation and collapse behavior
- Search debounce timing
- Table sorting (alphabetical default is fine)

</decisions>

<specifics>
## Specific Ideas

- Catalog page is intentionally simple — each tab is the same pattern (list of names with inline CRUD)
- Supplier form follows standard admin form patterns (separate page, back button, save/cancel)
- The sidebar navigation is being introduced in this phase and will grow as future phases are built

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BaseEntity` (id, created_at, updated_at): All new entities extend this
- `@Roles(Role.ADMIN)` decorator: Write endpoints restricted to ADMIN
- `@Public()` decorator: For any routes that skip auth
- Shadcn/ui components: Tabs, Card, Input, Button, Label, Select, Separator, Avatar, DropdownMenu
- `api.ts` fetch wrapper: Authenticated requests to backend
- `useIsAdmin` hook: Frontend role check for conditional rendering
- `Header` component: Will be modified to integrate with new sidebar layout
- `react-hook-form` + `zod`: Available for supplier form validation
- `sonner` (Toaster): Toast notifications already wired in layout

### Established Patterns
- Backend: NestJS module (controller + service + entity + DTO + module) registered in AppModule
- Backend: TypeORM entities extend BaseEntity, migrations-only (never synchronize)
- Backend: Swagger decorators on all controllers (@ApiTags, @ApiBearerAuth, @ApiOperation, @ApiResponse)
- Backend: Error messages in Spanish ("Email ya registrado")
- Frontend: Next.js App Router, pages in /app/*, components in /components/
- Frontend: Path alias @/* for imports, PascalCase components, camelCase hooks
- Frontend: Tailwind CSS v4 + Shadcn/ui new-york style

### Integration Points
- `AppModule.imports` array: Register new CatalogsModule and SuppliersModule
- `layout.tsx`: Header component will need rework to integrate sidebar
- Frontend routing: New pages at `/catalogos` and `/proveedores/*`
- `proxy.ts`: May need route protection updates for new pages

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 03-catalogs-and-suppliers*
*Context gathered: 2026-03-01*
