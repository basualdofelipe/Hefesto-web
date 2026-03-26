# Requirements: Nemea

**Defined:** 2026-02-28
**Core Value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automáticamente cuando cambian los precios de los insumos.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Authentication

- [x] **AUTH-01**: User can log in with Google OAuth and stay logged in across sessions
- [x] **AUTH-02**: Only whitelisted emails in DB can access the app (no public registration)
- [x] **AUTH-03**: User with ADMIN role can create, edit, and delete all resources
- [x] **AUTH-04**: User with USER role can view all resources but cannot modify anything
- [x] **AUTH-05**: Backend validates JWT from NextAuth on every request and checks email whitelist

### Catalogs

- [x] **CATL-01**: Admin can CRUD product types (ej: Billetera, Cinturón)
- [x] **CATL-02**: Admin can CRUD product names (ej: Hefesto, Ares)
- [x] **CATL-03**: Admin can CRUD product finishes (ej: Lisa, Grabada)
- [x] **CATL-04**: Admin can CRUD product colors (ej: Marrón, Negro)
- [x] **CATL-05**: Admin can CRUD product sizes (ej: Chico, Grande)
- [x] **CATL-06**: Admin can CRUD supply types (ej: Cuero, Herraje, Packaging)

### Suppliers

- [x] **SUPP-01**: Admin can CRUD suppliers with name, address, email, phone, whatsapp, description
- [x] **SUPP-02**: Admin can deactivate/reactivate suppliers (soft delete)

### Supplies & Prices

- [x] **SPPL-01**: Admin can CRUD supplies with name, type, and supplier
- [x] **SPPL-02**: Admin can add a new price to a supply (creates historical record, previous prices preserved)
- [x] **SPPL-03**: User can view price history for any supply
- [x] **SPPL-04**: Current price of a supply is always the most recent record in history
- [x] **SPPL-05**: Admin can deactivate/reactivate supplies (soft delete)

### Products

- [x] **PROD-01**: Admin can create a product by selecting type, name, finish, color, and size
- [x] **PROD-02**: Each product has an auto-generated unique SKU (sku_code)
- [x] **PROD-03**: Admin can edit product attributes
- [x] **PROD-04**: Admin can deactivate/reactivate products (soft delete)
- [x] **PROD-05**: Admin can define the material composition (BOM) of a product: list of supplies with quantity and unit_type (m2, unit, meter, kg)
- [x] **PROD-06**: When BOM changes, old composition is preserved as history (is_active flag)
- [x] **PROD-07**: Admin can add a selling price to a product (historical record)

### Cost Calculation

- [x] **COST-01**: System calculates product cost dynamically: SUM(latest_price(supply) × quantity) for all active materials
- [x] **COST-02**: Product list shows calculated cost per product
- [x] **COST-03**: When a supply price is updated, all products using that supply reflect the new cost automatically (no manual recalculation)
- [x] **COST-04**: Product detail shows cost breakdown by material

### Expenses

- [x] **EXPN-01**: Admin can register an expense with amount, concept, date, and category
- [x] **EXPN-02**: Admin can view list of expenses filtered by category or date
- [x] **EXPN-03**: Categories include: materia prima, packaging, envío, herramientas, servicios, otros

### Infrastructure

- [x] **INFR-01**: Backend runs on NestJS 11 with TypeORM and PostgreSQL
- [x] **INFR-02**: Database uses migrations only (never synchronize)
- [x] **INFR-03**: Docker Compose for local PostgreSQL development
- [x] **INFR-04**: All tables have created_at and updated_at timestamps
- [x] **INFR-05**: TypeScript strict mode, no any, explicit return types

## v1.1 Requirements

Requirements for milestone v1.1: Tiendanube & Investor Dashboard.

### Hardening

- [ ] **HARD-01**: Next.js middleware protects all app routes — unauthenticated users redirected to /login
- [ ] **HARD-02**: apiClientFetch detects 401 responses and redirects to login (JWT expiry handling)
- [ ] **HARD-03**: Backend endpoints enforce role-based access via @Roles decorators (ADMIN for mutations, USER for read-only)
- [ ] **HARD-04**: Admin can manage users (create, view, deactivate) from a /usuarios page in the frontend
- [ ] **HARD-05**: Acceso-denegado page shows clear message for unauthorized users
- [ ] **HARD-06**: Supply type "producción externa" exists as a catalog seed for taller-sourced products
- [ ] **HARD-07**: Shared types extracted (SupplyOption, UNIT_LABELS, formatDate, cleanSupplierData, SupplyCombobox) — zero duplication

### Product UX

- [ ] **PRUX-01**: Product table groups hierarchically: type → name → finish, with collapsible sections
- [ ] **PRUX-02**: BOM group editor scopes to name level (all "Hércules") instead of type level (all "Cinturones")
- [ ] **PRUX-03**: Individual products can have custom BOM overrides that differ from the group default

### Tiendanube Config

- [ ] **TNCF-01**: Admin can view and edit Tiendanube plans (Inicial/Esencial/Impulso/Escala) with rates per payment method and deposit timing
- [ ] **TNCF-02**: Admin can view and edit installment fee rates (1, 3, 6, 9, 12 cuotas)
- [ ] **TNCF-03**: Admin can view and edit tax config (IVA rate, IIBB alícuota, Tiendanube transaction fee per plan)
- [ ] **TNCF-04**: Config page includes a "Verificar tasas" link to the official Pago Nube rates page

### Calculadora

- [ ] **CALC-01**: User can select a product and see net profit calculated with real cost + Tiendanube deductions (forward mode)
- [ ] **CALC-02**: User can enter a desired profit and see the required selling price (inverse mode)
- [ ] **CALC-03**: Calculadora uses product costs from DB and Tiendanube config from admin-editable tables
- [ ] **CALC-04**: Batch calculation endpoint returns margins for all products in the catalog

### Simulator

- [ ] **SCEN-01**: User can create a named scenario with overridden selling prices for specific products
- [ ] **SCEN-02**: User can apply bulk price adjustments (e.g., "+10% all cinturones", "-20% entire catalog")
- [ ] **SCEN-03**: Scenario shows recalculated margins for all affected products using real costs + overridden prices
- [ ] **SCEN-04**: Scenarios are user-scoped (each investor sees only their own) and persist in DB

### Dashboard

- [ ] **DASH-01**: Investor can view a catalog summary table showing all products with cost, selling price, and net margin
- [ ] **DASH-02**: Dashboard can be filtered by product type
- [ ] **DASH-03**: Dashboard shows aggregate metrics (average margin, total catalog value, best/worst margin products)

## v2+ Requirements

Deferred to future release. Tracked but not in current roadmap.

### Tiendanube API Integration

- **TAPI-01**: Connect to Tiendanube API to read actual sales data
- **TAPI-02**: Cross-check calculated commissions vs actual charges from Tiendanube
- **TAPI-03**: Track sale prices, discounts, and multi-product orders from Tiendanube

### B2B

- **B2B-01**: Admin can manage B2B clients with addresses
- **B2B-02**: Admin can create orders with products and quantities
- **B2B-03**: Orders have status tracking (pendiente, enviado, entregado, cancelado)
- **B2B-04**: Admin can manage shippers and shipping methods

### Workflows

- **WKFL-01**: Admin can create purchase approval requests
- **WKFL-02**: Investors receive approval requests by email
- **WKFL-03**: Investors can approve/reject purchase requests

### Data Migration

- **MIGR-01**: Import existing product data from Google Sheets
- **MIGR-02**: Import existing supply and price data from Google Sheets

## Out of Scope

| Feature | Reason |
|---------|--------|
| Stock quantity tracking | No warehouse, no physical stock movements, no fulfillment process |
| Labor cost in BOM | No time-tracking infrastructure, materials-only cost model |
| Barcode/QR scanning | Single admin enters data via web form, no warehouse workflow |
| AI demand forecasting | No sales data in v1, nothing to forecast from |
| Multi-currency prices | Business operates in ARS, USD supplies entered at ARS equivalent |
| Real-time collaboration | 2-3 users total, standard REST is sufficient |
| Automated purchase orders | No stock levels tracked = no reorder point to trigger |
| Mobile app | Web-first, responsive design covers mobile access |
| Multi-idioma | App interna en español |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFR-01 | Phase 1: Backend Scaffold | Complete (01-01) |
| INFR-02 | Phase 1: Backend Scaffold | Complete (01-02) |
| INFR-03 | Phase 1: Backend Scaffold | Complete (01-02) |
| INFR-04 | Phase 1: Backend Scaffold | Complete (01-02) |
| INFR-05 | Phase 1: Backend Scaffold | Complete (01-01) |
| AUTH-01 | Phase 2: Auth | Complete (02-03, E2E verified) |
| AUTH-02 | Phase 2: Auth | Complete (02-01) |
| AUTH-03 | Phase 2: Auth | Complete (02-01) |
| AUTH-04 | Phase 2: Auth | Complete (02-01) |
| AUTH-05 | Phase 2: Auth | Complete (02-01) |
| CATL-01 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| CATL-02 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| CATL-03 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| CATL-04 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| CATL-05 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| CATL-06 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| SUPP-01 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| SUPP-02 | Phase 3: Catalogs and Suppliers | Complete (03-02 API + 03-03 UI) |
| SPPL-01 | Phase 4: Supplies and Price History | Complete |
| SPPL-02 | Phase 4: Supplies and Price History | Complete |
| SPPL-03 | Phase 4: Supplies and Price History | Complete |
| SPPL-04 | Phase 4: Supplies and Price History | Complete |
| SPPL-05 | Phase 4: Supplies and Price History | Complete |
| PROD-01 | Phase 5: Products and BOM | Complete |
| PROD-02 | Phase 5: Products and BOM | Complete |
| PROD-03 | Phase 5: Products and BOM | Complete |
| PROD-04 | Phase 5: Products and BOM | Complete |
| PROD-05 | Phase 5: Products and BOM | Complete |
| PROD-06 | Phase 5: Products and BOM | Complete |
| PROD-07 | Phase 5: Products and BOM | Complete |
| COST-01 | Phase 6: Cost Calculation | Complete |
| COST-02 | Phase 6: Cost Calculation | Complete |
| COST-03 | Phase 6: Cost Calculation | Complete |
| COST-04 | Phase 6: Cost Calculation | Complete |
| EXPN-01 | Phase 7: Expenses | Complete |
| EXPN-02 | Phase 7: Expenses | Complete |
| EXPN-03 | Phase 7: Expenses | Complete |

**Coverage:**
- v1 requirements: 37 total (CONF-01, CONF-02 moved to v2)
- Mapped to phases: 37
- Unmapped: 0

---
*Requirements defined: 2026-02-28*
*Last updated: 2026-02-28 — traceability populated by roadmapper*
