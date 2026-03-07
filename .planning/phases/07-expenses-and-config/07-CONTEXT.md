# Phase 7: Expenses - Context

**Gathered:** 2026-03-06
**Status:** Ready for planning

<domain>
## Phase Boundary

Expense tracking at parity with the Google Sheets. Admin registers expenses with amount, concept, date, and category. Expense list grouped by month with filters by category and date range. Expense categories are editable in DB (not hardcoded enum). Tiendanube config (CONF-01, CONF-02) removed from v1 — deferred to v2 alongside the calculadora.

</domain>

<decisions>
## Implementation Decisions

### Expense list layout
- Grouped by month (e.g., "Marzo 2026") with collapsible sections
- Each month header shows: month name + expense count + subtotal (e.g., "Marzo 2026 (5 gastos) — Total: $45.200")
- Table columns per row: Fecha, Concepto, Categoria (colored badge), Monto
- Summary bar at top showing grand total + count + filtered period — updates dynamically with filters
- Default view: current month only (user expands range with date picker)

### Filtering
- Category dropdown filter (one category or "Todas")
- Date range picker (desde/hasta)
- Both filters combine — summary bar and month groups update accordingly

### Category badges
- Each of the 6 categories gets a distinct colored badge in the table
- Colors are Claude's discretion — just make them visually distinguishable

### Create/edit expense
- Modal/dialog over the list (same pattern as supply create in Phase 4)
- "+ Nuevo gasto" button opens dialog
- Fields: monto (required), concepto (required, free text), fecha (required, defaults to today), categoria (required, dropdown select)
- Edit: same modal prefilled with existing values — click edit action on row
- Date defaults to today on create, admin can change for past expenses

### Delete expense
- Hard delete from DB (not soft delete)
- Delete button on each row opens confirmation dialog ("Eliminar este gasto?")
- No undo — once confirmed, the record is removed

### Expense categories as editable catalog
- Categories stored in DB as a catalog entity (like product_type, supply_type)
- NOT a hardcoded enum — admin can add/edit/delete categories
- New tab "Categorias de Gasto" added to the existing /catalogos page
- Same inline CRUD pattern as other catalog tabs (click to edit, inline add)
- Seeded with 6 initial categories: materia prima, packaging, envio, herramientas, servicios, otros
- Block deletion if category is used by existing expenses (same as other catalogs)

### Navigation
- New sidebar group "Finanzas" with "Gastos" as its first item
- Sidebar order: Inicio > Finanzas (Gastos) > Productos > Datos base (Catalogos, Proveedores, Insumos)
- Route: /finanzas/gastos (nested to allow future items like /finanzas/reportes)

### Roles
- ADMIN: full CRUD on expenses and expense categories
- USER: read-only access to expense list (view, filter, but no create/edit/delete)
- Same pattern as all other modules

### Tiendanube config — REMOVED from v1
- CONF-01 and CONF-02 moved to v2 requirements
- Config will be implemented alongside the calculadora Tiendanube
- Success criteria 3 and 4 (config view/edit and USER read-only on config) eliminated from this phase

### Claude's Discretion
- Exact styling of month group headers and collapsible animations
- Modal size and layout for expense create/edit
- Category badge color assignments
- Loading and error states
- Toast messages and duration
- Date picker component choice and behavior
- Empty state design for expenses list
- Summary bar styling and layout

</decisions>

<specifics>
## Specific Ideas

- Month grouping mirrors the mental model of "cuanto gaste este mes" — matches how the Google Sheets was used
- Categories as editable catalog future-proofs for when the business adds new expense types without needing code changes
- /finanzas/gastos route anticipates future finance-related pages (reportes, dashboard inversores in v2)
- Hard delete for expenses is intentional — these are operational records, not audit trails. If entered wrong, just delete and re-enter.

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BaseEntity` (id UUID, created_at, updated_at): Expense entity extends this
- `@Roles(Role.ADMIN)` decorator: Write endpoints restricted to ADMIN
- Shadcn/ui: Dialog, Table, Input, Button, Select, Collapsible, Badge components
- `apiClientFetch` wrapper: For mutations from client components
- `react-hook-form` + `zod`: For expense form validation in modal
- `sonner` Toaster: Toast notifications already wired
- Catalog inline CRUD pattern: Reuse for expense categories tab
- Grouped table pattern (supplies by type): Adapt for expenses by month

### Established Patterns
- Backend: NestJS module (controller + service + entity + DTO + module) registered in AppModule
- Backend: TypeORM migrations-only, Swagger decorators, Spanish error messages
- Backend: Catalog pattern (simple name entity + CRUD + block delete if referenced)
- Frontend: Server-component data fetching + client-component interactivity
- Frontend: Modal for create/edit (supply pattern), confirmation dialog for destructive actions
- Frontend: Collapsible grouped sections (supply type groups)
- Frontend: Catalog tabs with inline editing (/catalogos page)

### Integration Points
- `AppModule.imports`: Register new ExpensesModule + ExpenseCategoriesModule (or within CatalogsModule)
- Catalogs frontend: Add new tab "Categorias de Gasto" to existing tabs
- `AppSidebar.tsx`: Add "Finanzas" group with "Gastos" item, reorder existing groups
- Frontend routing: New route group at /finanzas/gastos/
- Backend config/ directory: Currently has env.validation.ts and typeorm.config.ts only (no collision with Tiendanube config)

</code_context>

<deferred>
## Deferred Ideas

- Tiendanube config (CONF-01, CONF-02) — v2, alongside calculadora
- Expense reports/analytics (totals by category over time, charts) — v2 dashboard
- Expense export to Excel/PDF — v2
- Recurring expenses (auto-create monthly) — v2+

</deferred>

---

*Phase: 07-expenses-and-config*
*Context gathered: 2026-03-06*
