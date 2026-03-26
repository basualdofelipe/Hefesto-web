# Feature Research: v1.1 Tiendanube & Investor Dashboard

**Domain:** E-commerce pricing simulation + investor margin analysis for artisan leather goods
**Researched:** 2026-03-26
**Confidence:** HIGH (core features drawn from existing prototype + verified Tiendanube rates + real business rules; market patterns confirmed via multiple sources)

---

## Context

This research covers NEW capabilities for v1.1. The v1.0 system (product CRUD, supply CRUD with price history, BOM, dynamic cost calculation, expenses, auth) is already built and deployed. The v1.1 milestone adds three capability clusters:

1. **Pricing simulation** -- Tiendanube config + calculadora (forward/inverse)
2. **Investor intelligence** -- margin summary dashboard + scenario simulator
3. **Hardening & UX** -- auth fixes, DRY cleanup, users admin, product hierarchy

The "users" remain the same: 1 admin (owner) and 1-2 investors (USER role, read-only + simulator). This is NOT a commercial SaaS product -- it serves a single business. "Table stakes" means: if this feature is missing, the v1.1 milestone fails to deliver its stated value.

**Key constraint:** The calculadora prototype already exists as a working React component (`_archive/calculadora/calculadora-tiendanube.jsx`). The forward/inverse calculation logic is proven. The migration is about connecting it to real DB data and making rates admin-editable, not inventing new math.

**Key constraint (SIRTAC):** As of March 2, 2026, Pago Nube applies SIRTAC (Sistema de Recaudacion sobre Tarjetas de Credito y Compra), a unified IIBB retention system. The retention percentage varies per vendor based on COMARB padron (up to 5%). The app already models this as a configurable `alicuotaIIBB` -- this is correct and sufficient. The rate is NOT deterministic from code; the admin must look it up and enter it.

---

## Feature Landscape

### Table Stakes (Must Have for v1.1 to Deliver Value)

These features define the milestone's stated goals. Missing any of them means v1.1 is incomplete.

| Feature | Why Expected | Complexity | Dependencies | Notes |
|---------|--------------|------------|--------------|-------|
| **Tiendanube config tables** | Admin must be able to update rates when Tiendanube changes pricing (happens periodically). Hardcoded rates become stale immediately | MEDIUM | Auth (ADMIN role), new DB tables | 4 plans x 3 withdrawal periods x card rates + transfer rates + installment rates + IIBB + IVA. All verified against official Tiendanube docs as of March 2026 |
| **Calculadora forward mode** | The core ask: "I sell this at $87,000 -- what do I actually keep after Tiendanube commissions, installment financing, IIBB, and IVA?" | MEDIUM | Tiendanube config, Cost calculation (v1.0), Product data | Logic exists in prototype. Migration means: pull product cost from DB, pull config from DB, run `calcForward()` |
| **Calculadora inverse mode** | "I want to profit $50,000 on this wallet -- what price do I need to set?" Uses binary search | LOW | Forward mode (reuses it in loop) | 100-iteration binary search over `calcForward()`. Already proven in prototype. Trivial once forward works |
| **Investor dashboard: catalog margin summary** | Investors need to see: all products, their real cost, selling price, and net margin. This is why they have access to the app | MEDIUM | Cost calculation, Product selling prices | A read-only table showing every active product with cost, selling price, and gross margin (price - cost). This is the "investor value" of the app |
| **Users admin page** | Backend CRUD for users exists (GET, POST, DELETE at `/users`). No frontend UI. Admin currently can't manage the whitelist without DB access | LOW | Auth (ADMIN), existing backend endpoints | Simple table + add/remove. Backend is done |
| **Auth hardening: middleware.ts** | Frontend routes are unprotected -- `proxy.ts` isn't wired as Next.js middleware. Any URL is accessible without auth | LOW | Existing auth system | Wire existing `proxy.ts` logic as `middleware.ts`. Low risk, high impact |
| **Auth hardening: 401 handling** | When JWT expires, API calls fail silently. User sees broken UI instead of redirect to login | LOW | Existing apiClientFetch | Add 401 interceptor in fetch wrapper. Redirect to `/login` on expiry |

### Differentiators (Beyond Basic Expectations)

These features go beyond "investor can see margins" and create genuine analytical value.

| Feature | Value Proposition | Complexity | Dependencies | Notes |
|---------|-------------------|------------|--------------|-------|
| **Scenario simulator with saved scenarios** | Investors can model "what if I raise all leather goods 10%" without touching real data. Named scenarios persist per user. This is unique for a tool at this scale -- most small-business cost tools don't have what-if modeling | HIGH | Investor dashboard, Cost calculation, Product data, new DB table | New `scenario` + `scenario_price_override` tables. Each scenario has a name, user_id, and a set of product price overrides. Dashboard re-calculates margins using override prices instead of real selling prices |
| **Product hierarchical grouping: type > name > finish** | Currently products are grouped flat by type. The ask is a tree: Billeteras > Hefesto > Lisa/Grabada > colors. Makes large catalogs navigable | MEDIUM | Existing product data model (5 dimensions already exist) | Frontend-only change. Data model already supports this -- products have type, name, finish, color, size. Just need nested collapsible UI instead of flat type grouping |
| **Calculadora connected to real product cost** | Instead of manually typing "costo producto: $6,534", the calculadora auto-fills the dynamic cost from the DB when you select a product. Eliminates human error | LOW | Cost calculation, Product CRUD, Tiendanube config | Product selector dropdown that fills `costoProducto` from the product's calculated cost. Simple but high-value UX improvement over the standalone prototype |
| **Net margin after Tiendanube deductions in dashboard** | Gross margin (price - cost) is nice. Net margin after commissions, IIBB, and IVA is the REAL number investors care about. Requires running `calcForward()` for each product | MEDIUM | Forward mode, Investor dashboard, Tiendanube config | For each product: run calcForward(sellingPrice, cost, plan, ...) to get `gananciaReal`. Show net margin alongside gross margin. This is the "killer column" in the dashboard |
| **"Verify rates" link to Pago Nube official page** | Tiendanube rates change. A link directly to the official rates page lets the admin check and update without searching | LOW | Tiendanube config page | Single anchor tag to `https://ayuda.tiendanube.com/es_AR/pago-nube-2/cuales-son-las-comisiones-de-pago-nube`. Trivial but thoughtful |
| **BOM group editor scoped to name (not type)** | Current group BOM editor applies to all products of a type. Should be scoped to product name (e.g., all "Hefesto" wallets share the same recipe, but "Ares" wallets have different materials) | MEDIUM | Existing BOM group editor, product hierarchy | Requires grouping by product_name_id instead of product_type_id. Logic change in the existing `BomGroupEditorDialog` |
| **Supply type "produccion externa"** | Business is pivoting from artisan production to buying finished goods from external workshops. Need a new supply type to track workshop costs | LOW | Existing supply type catalog | Add a new row to `supply_type` seed data. Maybe add a UI indicator. The BOM system already supports arbitrary supply types |
| **DRY cleanup** | 8 duplicate `SupplyOption` interfaces, 5 duplicate `UNIT_LABELS` constants, 4 duplicate `formatDate` functions across frontend. Technical debt that slows future work | LOW | None -- refactoring only | Extract shared types to `@/types/`, shared constants to `@/lib/constants.ts`, shared formatters to `@/lib/format.ts` |

### Anti-Features (Explicitly NOT Building)

Features that seem logical for v1.1 but would be wrong to include.

| Feature | Why It Seems Good | Why Problematic | Alternative |
|---------|-------------------|-----------------|-------------|
| **Real-time Tiendanube API integration** | Pull actual sales data from Tiendanube instead of simulating | Requires OAuth with Tiendanube API, webhook infrastructure, sales data parsing. Massive scope increase. The business doesn't need this -- the simulaton is for pricing decisions, not accounting | Keep the simulator using configured rates + manual selling prices. API integration is v2+ after the app proves its value |
| **Auto-updating rates from Tiendanube** | Scrape or poll Tiendanube for rate changes | Rates are behind a login wall, no public API for rates. Scraping is fragile and terms-violating. Rate changes happen a few times per year | Admin-editable config table + "verify rates" link. 2 minutes of manual work beats weeks of fragile automation |
| **Multi-scenario comparison side-by-side** | Like Cube/Runway, show two scenarios next to each other | This is FP&A software territory. The use case is 1 admin + 1-2 investors looking at 1 scenario at a time. Side-by-side adds significant UI complexity for a feature that would be used rarely | Save scenarios with names. View one at a time. Compare mentally or switch between them. If side-by-side becomes requested, add it in v1.2 |
| **Scenario sharing between users** | Investor A shares their scenario with Investor B | There are 1-2 investors total. They can just name their scenario descriptively. Building a sharing/permissions system for 2 users is absurd | Scenarios are per-user. Admin can see all scenarios. If an investor wants to show something, they screenshot it |
| **Historical margin tracking over time** | "Show me how margins changed in the last 6 months" | Requires snapshoting product costs and selling prices over time. The price history tables exist but building a time-series margin view is a new analytical dimension. High complexity for unclear value at this stage | The building blocks exist (price history, cost history). Time-series margin dashboard is a v2 feature once there's enough data to make it useful |
| **PDF report export for investors** | Generate a polished PDF with margins and scenarios | PDF generation (puppeteer/react-pdf) adds significant dependency and maintenance. The dashboard itself IS the report. For 1-2 investors who already have app access, there's no need for offline reports | Investors view the dashboard in-app. If offline access is needed, browser print-to-PDF works. Dedicated PDF generation is v2+ |
| **Batch price adjustment tool** | "Raise all prices by 10% with one click" | This writes real data -- changes actual selling prices. Dangerous in a system where price history matters. Conflates simulation (what-if) with action (actually changing prices). Also the admin already has per-product price editing | Scenarios serve the "what if" use case safely. Actual price changes remain deliberate, per-product actions |
| **Detailed fiscal breakdown per product** | Show exact IIBB retention, IVA debito/credito per product line | The IIBB rate via SIRTAC varies per transaction based on buyer location. Cannot be pre-calculated per-product accurately. The app explicitly declares fiscal values as "approximate" | Use a single configurable IIBB rate as an estimate. The admin consults with their accountant for exact fiscal planning. The app is a pricing tool, not an accounting system |
| **Expense allocation to products** | Assign overhead expenses (rent, packaging supplies) proportionally to products to get "fully loaded" cost | Changes the fundamental cost model from "direct materials only" to "absorption costing". Requires allocation methodology decisions, dramatically increases complexity | Keep expenses as a separate tracking module. Product cost = direct materials only. If needed, show total expenses as a sidebar metric on the dashboard so investors can mentally factor them in |

---

## Feature Dependencies

```
[Auth hardening: middleware.ts + 401 handling]
    (prerequisite for all v1.1 features -- broken auth undermines everything)

[DRY cleanup]
    (no dependencies, can happen in parallel, makes everything after it cleaner)

[Users admin page]
    └──requires──> existing backend CRUD (already built)
    (standalone, no feature dependencies)

[Tiendanube config tables]
    └──requires──> Auth (ADMIN-only editing) ← already built
    └──required by──> Calculadora forward mode
    └──required by──> Calculadora inverse mode
    └──required by──> Net margin after Tiendanube deductions (dashboard)

[Calculadora forward mode]
    └──requires──> Tiendanube config tables
    └──requires──> Cost calculation (v1.0) ← already built
    └──required by──> Calculadora inverse mode (calls forward in loop)
    └──required by──> Net margin column in dashboard

[Calculadora inverse mode]
    └──requires──> Calculadora forward mode
    (standalone otherwise)

[Investor dashboard: catalog margin summary]
    └──requires──> Cost calculation (v1.0) ← already built
    └──requires──> Product selling prices (v1.0) ← already built
    └──required by──> Scenario simulator
    └──required by──> Net margin column

[Net margin after Tiendanube deductions]
    └──requires──> Calculadora forward mode
    └──requires──> Investor dashboard
    └──enhances──> Dashboard from "basic gross margin" to "real profit visibility"

[Scenario simulator]
    └──requires──> Investor dashboard (it's a layer ON TOP of the dashboard)
    └──requires──> Cost calculation
    └──requires──> new DB tables (scenario, scenario_price_override)
    (most complex new feature)

[Product hierarchy: type > name > finish]
    └──requires──> existing product data ← already built
    └──enhances──> Investor dashboard (better navigation)
    └──enhances──> Calculadora (better product selection)
    (frontend-only, no backend changes)

[BOM group editor scoped to name]
    └──requires──> existing BOM group editor ← already built
    └──enhances──> Product hierarchy view
    (scoped refactoring)

[Supply type "produccion externa"]
    └──requires──> existing supply type catalog ← already built
    (seed data + possibly UI indicator)
```

### Dependency Notes

- **Auth hardening must come first:** Every feature relies on working auth. A broken JWT expiry path or unprotected routes undermine the entire app.
- **Tiendanube config is the gateway:** Both the calculadora and the net margin dashboard column require configured rates. This is the single most important new backend entity.
- **Investor dashboard before scenario simulator:** The simulator is a layer that overrides selling prices on the dashboard. You need the base dashboard first.
- **Forward mode before inverse mode:** Inverse literally calls forward in a loop. This is an unbreakable dependency.
- **Product hierarchy is independent:** Pure frontend refactoring. Can happen in parallel with backend work.
- **DRY cleanup is independent:** Technical debt cleanup with no feature dependencies. Best done early to simplify all subsequent work.

---

## Milestone Definition

### v1.1 Core (Must Ship)

These features together fulfill the milestone's stated goals.

- [ ] **Auth hardening** (middleware + 401 handling) -- foundation for everything else
- [ ] **DRY cleanup** -- reduces friction for all subsequent frontend work
- [ ] **Users admin page** -- unblocks admin from needing DB access for whitelist
- [ ] **Tiendanube config tables** -- rates stored in DB, admin-editable
- [ ] **Calculadora forward mode** -- price to profit after all deductions
- [ ] **Calculadora inverse mode** -- desired profit to required price
- [ ] **Investor dashboard: catalog margin summary** -- all products with cost, price, margin
- [ ] **Scenario simulator** -- named what-if scenarios with price overrides

### v1.1 Polish (Ship If Time Allows)

These features enhance the milestone but aren't blocking.

- [ ] **Net margin after Tiendanube deductions in dashboard** -- the "killer column" that shows real profit
- [ ] **Product hierarchy view** -- type > name > finish tree navigation
- [ ] **Calculadora connected to real product cost** -- auto-fill cost from DB
- [ ] **BOM group editor scoped to name** -- correct scoping for the business model
- [ ] **Supply type "produccion externa"** -- supports workshop pivot
- [ ] **"Verify rates" link** -- convenience feature

### Defer to v1.2+

Explicitly out of scope for this milestone.

- [ ] **Historical margin tracking** -- needs time-series data accumulation first
- [ ] **PDF export** -- browser print-to-PDF is sufficient for 1-2 investors
- [ ] **Multi-scenario comparison** -- view one at a time is adequate for the user count
- [ ] **Tiendanube API integration** -- massive scope, unclear value vs manual config

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority | Phase Suggestion |
|---------|------------|---------------------|----------|------------------|
| Auth hardening (middleware + 401) | HIGH | LOW | P1 | Hardening phase (early) |
| DRY cleanup | MEDIUM | LOW | P1 | Hardening phase (early) |
| Users admin page | MEDIUM | LOW | P1 | Hardening phase (early) |
| Tiendanube config tables (backend + frontend) | HIGH | MEDIUM | P1 | Config phase |
| Calculadora forward mode | HIGH | MEDIUM | P1 | Calculator phase |
| Calculadora inverse mode | HIGH | LOW | P1 | Calculator phase (after forward) |
| Investor dashboard: margin summary | HIGH | MEDIUM | P1 | Dashboard phase |
| Scenario simulator | HIGH | HIGH | P1 | Dashboard phase (after base dashboard) |
| Net margin after TN deductions | HIGH | MEDIUM | P2 | Dashboard phase (enhancement) |
| Product hierarchy view | MEDIUM | MEDIUM | P2 | UX phase or parallel |
| Calculadora with real product cost | MEDIUM | LOW | P2 | Calculator phase (enhancement) |
| BOM group editor scoping | MEDIUM | MEDIUM | P2 | UX phase |
| Supply type produccion externa | LOW | LOW | P2 | Hardening or catalog phase |
| Verify rates link | LOW | LOW | P2 | Config phase (trivial addition) |

**Priority key:**
- P1: Must ship in v1.1 -- defines the milestone
- P2: Should ship if time allows -- valuable but not milestone-defining

---

## Feature Detail: Tiendanube Config

**Verified data model** (confirmed against official Tiendanube docs, March 2026):

| Plan | Card 1d | Card 7d | Card 14d | Transfer | TN Tx Fee (bonified) |
|------|---------|---------|----------|----------|---------------------|
| Inicial | 7.69% | 5.59% | 4.69% | 1.50% | 2.0% (bonified) |
| Esencial | 6.09% | 4.39% | 3.49% | 1.50% | 1.5% (bonified) |
| Impulso | 5.89% | 4.19% | 3.29% | 0.99% | 1.0% (bonified) |
| Escala | 5.59% | 3.89% | 2.99% | 0.85% | 0.7% (bonified) |

**Installment financing rates** (seller absorbs these for "cuotas sin interes"):
- 1 cuota: 0%
- 3 cuotas: 8.42%
- 6 cuotas: 17.41%
- 9 cuotas: 27.04%
- 12 cuotas: 37.39%

**Additional config fields:**
- IVA rate: 21% (configurable in case it changes)
- IIBB/SIRTAC aliquot: ~2.5% (varies per vendor, must be manually configured)

**Confidence:** HIGH -- rates verified against official Tiendanube help center AND match the existing prototype's hardcoded values exactly.

**DB design recommendation:** Use a structured approach rather than a flat key-value config table. Two tables:
1. `tiendanube_plan` -- one row per plan with card rates per withdrawal period + transfer rate + tx fee
2. `tiendanube_config` -- singleton row with IVA rate, IIBB rate, installment rates (JSONB or separate columns), active plan selection

This makes the admin UI straightforward: one form for global settings, one table for plan rates.

---

## Feature Detail: Calculadora Tiendanube

**Forward mode formula** (verified against prototype + business rules):

```
1. totalCliente     = precioVenta + costoEnvio
2. tasaConIVA       = tasaBase * 1.21
3. comisionPagoNube = totalCliente * (tasaConIVA / 100)
4. costoFinanciacion = totalCliente * (tasaCuotas / 100)
5. baseGravada      = totalCliente / 1.21
6. ivaDebito        = totalCliente - baseGravada
7. ivaCreditoProducto  = costoProducto * 0.21
8. ivaCreditoComision  = comisionPagoNube * (0.21 / 1.21)
9. ivaNeto          = ivaDebito - ivaCreditoProducto - ivaCreditoComision
10. retencionIIBB   = totalCliente * (alicuotaIIBB / 100)
11. netoRecibido    = totalCliente - comisionPagoNube - costoFinanciacion - retencionIIBB
12. costoProductoConIVA = costoProducto * 1.21
13. gananciaReal    = netoRecibido - costoProductoConIVA - ivaNeto
14. margen          = (gananciaReal / precioVenta) * 100
```

**Inverse mode:** Binary search (100 iterations) over `calcForward()`. Range: `[costoProducto, costoProducto * 20]`. Converges to the selling price that yields the desired profit.

**Test case** (from business rules, must pass):
- Precio venta: $87,000 / Envio: $7,314.99 / Costo: $6,534.48
- Expected: neto recibido ~$78,489, ganancia real calculable

**Where to run this calculation:** Frontend. The calculation is deterministic, uses only config values + user inputs, and needs to be reactive (recalculate on every input change). No reason to send an API request for each keystroke. Fetch config once, calculate in the browser.

**Confidence:** HIGH -- logic is proven in the prototype, formulas match business rules doc.

---

## Feature Detail: Investor Dashboard

**What investors actually need to see** (based on project context and market research):

The investor dashboard is a **read-only catalog summary** showing:

| Column | Source | Notes |
|--------|--------|-------|
| Producto | product name (type + name + finish + color + size) | Grouped hierarchically if product hierarchy is implemented |
| SKU | product.skuCode | For reference |
| Costo | Dynamic cost calculation (v1.0) | SUM(latest_price * quantity) for all BOM items |
| Precio de venta | Latest selling price from product_price_history | What it's listed at on Tiendanube |
| Margen bruto | (precio - costo) / precio * 100 | Simple gross margin |
| Ganancia neta TN | calcForward(precio, costo, ...).gananciaReal | Real profit after Tiendanube takes its cut (P2 enhancement) |
| Margen neto TN | gananciaReal / precio * 100 | Real margin percentage (P2 enhancement) |
| Estado | isActive badge | Active/Inactive indicator |

**Aggregation rows:**
- Per product type: average margin, total cost, etc.
- Grand total: catalog-wide average margin, sum of costs

**Key UX decisions:**
- Default sort: by product type, then by name (matches how the business thinks about the catalog)
- Filter: active only by default, toggle to show inactive
- No editing: this is a read-only view. Price changes happen on the product page
- Admin sees the same dashboard as investors (plus nav to admin pages)

**Confidence:** HIGH -- this is a straightforward data display. The building blocks (product data, costs, selling prices) all exist.

---

## Feature Detail: Scenario Simulator

**How scenario simulation should work** (synthesized from FP&A tool patterns adapted to Nemea's scale):

### Core Concept
A scenario is a named set of selling price overrides. When viewing a scenario, the dashboard shows what margins WOULD BE if those prices were used, without changing any real data.

### Data Model
```
scenario
  - id (UUID)
  - name (VARCHAR, e.g., "Aumento cuero 10% - Abril 2026")
  - description (TEXT, nullable)
  - userId (FK to users) -- each user owns their scenarios
  - createdAt, updatedAt

scenario_price_override
  - id (UUID)
  - scenarioId (FK to scenario)
  - productId (FK to product)
  - overridePrice (DECIMAL) -- the hypothetical selling price
```

### UX Flow
1. **Investor opens dashboard** -- sees real data (no scenario active)
2. **Creates new scenario** -- gives it a name, e.g., "Suba 10% cuero"
3. **Overrides prices** -- for each product (or group), enters the hypothetical selling price
4. **Views scenario** -- dashboard recalculates all margins using override prices where present, real prices for everything else
5. **Saves scenario** -- persists for future reference
6. **Switches back to "real"** -- toggles scenario off, sees actual data again

### Bulk Override UX
The most common scenario is "raise all X by Y%". Instead of editing each product individually:
- Select products by type or name group
- Enter a percentage increase/decrease
- System applies the math and fills in override prices

This is NOT a batch price change. It only creates override entries in the scenario.

### Expected UX Pattern
Based on patterns from Cube, Runway, and Finmark (adapted for 1-2 users instead of finance teams):
- Dropdown/toggle at the top of the dashboard: "Real data" vs "Scenario: [name]"
- When a scenario is active, show a colored banner: "Viewing scenario: Suba 10% cuero"
- Override cells highlighted differently (e.g., blue background) so it's clear which prices are real vs hypothetical
- Simple CRUD for scenarios: create, rename, duplicate, delete

**Confidence:** MEDIUM -- the UX patterns are well-established in FP&A tools, but adapting them to a 2-user leather goods app requires judgment calls. The data model is straightforward.

---

## Feature Detail: Product Hierarchy

**Current state:** Products are grouped by `product_type` (Billeteras, Cinturones, Deskpads) with a flat list within each group.

**Desired state:** Three-level hierarchy:
```
Billeteras (type)
  └── Hefesto (name)
       ├── Lisa (finish)
       │    ├── Marron (color/size)
       │    └── Negro
       └── Grabada (finish)
            └── Marron
  └── Ares (name)
       └── Lisa
            └── Negro
```

**Implementation approach:** Frontend-only. No backend changes needed. The product data already has type, name, finish, color, and size as separate dimensions. The frontend currently groups by `product.type.id`. Extend to nest by `product.name.id` within type, and optionally by `product.finish.id` within name.

**Shadcn tree view:** A [shadcn-tree-view](https://github.com/MrLightful/shadcn-tree-view) community component exists, but may be overkill. Nested collapsible sections using the existing `Collapsible` Radix primitive (already in the project via Shadcn sidebar) are simpler and consistent with the current design.

**Confidence:** HIGH -- pure frontend work with existing data. No unknowns.

---

## Feature Detail: Users Admin Page

**Current state:**
- Backend: `GET /users`, `POST /users`, `DELETE /users/:id` endpoints exist (ADMIN only)
- Frontend: No page exists. Admin must use DB to manage whitelist

**Required UI:**
- Table showing: email, name, role, avatar (from Google), createdAt
- Add user: form with email + role selector (ADMIN/USER)
- Remove user: delete with confirmation dialog
- Edit role: toggle ADMIN/USER (optional, nice to have)

**No need for:** password management, profile editing, avatar upload, activity log, session management. These are Google OAuth users with fixed roles.

**Confidence:** HIGH -- backend is done, frontend is standard CRUD table.

---

## Competitor Feature Analysis (v1.1 specific)

| Feature | Craftybase (artisan) | Generic FP&A (Cube/Finmark) | Nemea v1.1 Approach |
|---------|----------------------|------------------------------|---------------------|
| Pricing calculator with marketplace fees | Yes (Etsy fee calculator) | N/A | Yes -- Tiendanube-specific with all Pago Nube commission tiers |
| Forward/inverse calculation | Forward only (cost + markup = price) | N/A | Both: price-to-profit AND profit-to-price (binary search) |
| What-if scenarios | No | Yes (multiple scenarios, side-by-side) | Yes, simplified: named scenarios with price overrides, one-at-a-time view |
| Investor dashboard | No (single-user tool) | Yes (investor reporting dashboards) | Yes -- read-only catalog margin summary with role-based access |
| Configurable marketplace rates | Etsy fees are hardcoded | N/A | Yes -- admin-editable config tables for all Tiendanube plans |
| Margin per product | Yes (material cost tracking) | Yes (P&L by product) | Yes -- gross margin now, net margin after TN deductions as P2 |
| Product hierarchy | Categories only | N/A | 3-level: type > name > finish tree view |

**Key differentiator:** No artisan-focused tool (Craftybase, Katana) offers inverse pricing calculation that accounts for marketplace commissions AND fiscal deductions (IVA, IIBB). Nemea's calculadora solves a real pain point unique to Argentine e-commerce on Tiendanube.

---

## Sources

**HIGH confidence (official/first-party):**
- Tiendanube commission rates: [Official help center](https://ayuda.tiendanube.com/es_AR/pago-nube-2/cuales-son-las-comisiones-de-pago-nube) -- verified March 2026
- Tiendanube SIRTAC/IIBB: [Costs and retentions](https://ayuda.tiendanube.com/es_AR/pago-nube-2/que-costos-debo-tener-en-cuenta-al-vender-con-pago-nube)
- SIRTAC official: [Argentina.gob.ar](https://www.argentina.gob.ar/economia/politicatributaria/armonizacion/sirtac)
- Nemea business rules: `.claude/docs/planning/business-rules.md`
- Nemea prototype: `_archive/calculadora/calculadora-tiendanube.jsx`
- Nemea PROJECT.md: `.planning/PROJECT.md`

**MEDIUM confidence (market research):**
- Craftybase pricing features: [craftybase.com/features/pricing-software](https://craftybase.com/features/pricing-software)
- Craftybase Etsy calculator: [craftybase.com/etsy/pricing-calculator](https://craftybase.com/etsy/pricing-calculator)
- Cube scenario analysis: [cubesoftware.com/scenario-what-if-analysis](https://www.cubesoftware.com/scenario-what-if-analysis)
- Finmark scenario planning: [finmark.com/features/scenario-planning](https://finmark.com/features/scenario-planning/)
- Runway FP&A: [runway.com](https://runway.com/)
- Investor dashboard best practices: [lucid.now/blog/investor-dashboards-features-best-practices](https://www.lucid.now/blog/investor-dashboards-features-best-practices/)
- Profit margin dashboards: [bizinfograph.com/dashboard-templates/profit-margin-dashboard](https://www.bizinfograph.com/dashboard-templates/profit-margin-dashboard)
- Tree view patterns: [primer.style/product/components/tree-view/guidelines](https://primer.style/product/components/tree-view/guidelines/)
- Shadcn tree view component: [github.com/MrLightful/shadcn-tree-view](https://github.com/MrLightful/shadcn-tree-view)

---

*Feature research for: v1.1 Tiendanube pricing simulation + investor dashboard*
*Researched: 2026-03-26*
