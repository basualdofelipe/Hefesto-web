# Feature Research

**Domain:** Product cost management / inventory system for artisan leather goods workshop
**Researched:** 2026-02-28
**Confidence:** HIGH (core features drawn from project's own business rules + medium confidence from market research on comparable tools like Craftybase, Katana MRP, MRPeasy)

---

## Context

Nemea replaces a Google Sheets + Apps Scripts system. The user's mental model is already formed: they know exactly what data they have and what calculations they need. This is not a greenfield discovery problem — the risk is building MORE than needed, not less. The scope below is calibrated to that reality.

The "users" of this system are: 1 admin (owner, full access) and 1–2 viewers (investors, read-only). The product does not compete in a market — it serves a single business. "Table stakes" in this context means: if this is missing, the Sheets replacement is worse than the original.

---

## Feature Landscape

### Table Stakes (Users Expect These)

These features define parity with the existing Google Sheets. Missing any of them means the app is objectively worse than what it replaces.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Supply CRUD with type and supplier | Core of the cost model — every product cost depends on supplies | LOW | supply, supply_type, suppliers tables already designed |
| Price history per supply | Prices change frequently (leather, hardware). History is auditable and the last price drives all costs | MEDIUM | One-append-only history table, query with ORDER BY date DESC LIMIT 1 |
| Dynamic cost calculation per product | Core value of the whole app: `SUM(last_price(supply) * quantity)`. Auto-updates when any supply price changes | MEDIUM | Calculated on-the-fly at query time. No cache needed at v1 scale (2–3 users, ~50 products) |
| Product CRUD with 5-dimension catalog | Products are defined as combinations: type → name → finish → color → size. Each combo = one SKU | MEDIUM | 5 catalog tables + product table + sku_code generation |
| Material composition per product (BOM) | Needed to calculate costs. Each product has a list of supplies with quantities and unit types (m2, unit) | MEDIUM | supplies_per_product_history with is_active flag |
| BOM version history | When you change what a product is made of, old compositions should be preserved for auditability | MEDIUM | is_active pattern on supplies_per_product_history — already in scope |
| Soft delete for products and supplies | An old product shouldn't be deleted when discontinued — it has cost history and BOM history | LOW | is_active boolean flag |
| Supplier CRUD with contact info | Need to know who supplies what, contact details for reordering | LOW | suppliers table: name, address, email, phone, whatsapp |
| Expense tracking with categories | The Sheets has this. Categories matter for understanding cost structure (raw material vs packaging vs shipping) | LOW | Simple expenses table with category enum |
| Selling price history per product | Prices to market change over time. Needed to see margin evolution | LOW | product_price_history table |
| Role-based access (admin vs viewer) | Investors see data but cannot modify. Owner controls everything | MEDIUM | Google OAuth whitelist + ADMIN/USER role enum |
| Authentication via Google OAuth | Internal app, 2–3 known users. No password management needed | MEDIUM | NextAuth (front) + JWT validation (back) + email whitelist in DB |

### Differentiators (Competitive Advantage)

These features go beyond parity with Sheets and create genuine value that Sheets cannot provide.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Automatic cost propagation | When supply price is updated, ALL product costs reflect it instantly without manual recalculation. Sheets required running a macro | LOW | Consequence of on-the-fly calculation design — free by architecture |
| SKU auto-generation | Sheets had a fragile Apps Script for this. A deterministic, reproducible SKU from the 5 product dimensions removes errors | LOW | Encode dimension IDs or abbreviations into a composable code |
| Cost breakdown by material type | See how much of a wallet's cost is leather vs hardware vs packaging. Sheets lumped it together | MEDIUM | Group BOM by supply_type when calculating — add type subtotals to cost response |
| BOM change tracking | Knowing when and why a product's composition changed. Sheets had no history | LOW | Already part of supplies_per_product_history with date + is_active — just needs a UI |
| Investor-friendly read-only view | Separate permission tier lets investors see product margins without any data modification risk | LOW | Role guard on mutation endpoints — already planned |
| Catalog management UI | In Sheets, adding a new product type required editing a lookup table manually. A CRUD UI for catalogs is substantially better | LOW | Standard CRUD for the 5 catalog tables + supply_type |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem useful for a cost/inventory app but are wrong for Nemea v1.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Stock quantity tracking (inventory levels) | Standard in inventory software | Nemea doesn't track physical stock. It tracks cost structure. There are no stock movements, no warehouse, no reorder points. Adding stock tracking would require a fundamentally different data model and processes that don't exist in the business today | Stick to cost management. Stock tracking is a v3+ feature when orders and fulfillment are implemented |
| Labor cost in BOM | Craftybase and similar tools include labor as a cost component | Nemea's current cost model is materials-only. Labor is not tracked in Sheets today. Introducing it requires time-tracking infrastructure and changes every single cost calculation | Track labor as a fixed overhead expense in the expenses module if needed. Don't bake it into BOM |
| Barcode / QR scanning | Typical inventory feature, feels modern | There's no warehouse, no receiving process, no physical stock movements. A single admin enters data via web form. Scanning adds hardware dependency with zero benefit | Manual data entry is fine for the scale (50–200 products, <5 users) |
| AI-driven demand forecasting | Trendy in 2025 inventory tools | No sales data, no order history, no connected sales channel in v1. There is nothing to forecast from | Connect Tiendanube API in v2 first, then consider forecasting in v3+ |
| Multi-currency support | Supplier prices might come in USD | Adds complexity to every price comparison and cost calculation. The business operates in ARS. USD-priced supplies can be entered as their ARS equivalent at purchase time | Enter all prices in ARS. If needed, add a currency field to supplies_price_history in v2 as a display annotation |
| Real-time collaboration / conflict resolution | Multiple users editing simultaneously | There are 2–3 users total, 1 is admin. Optimistic locking or last-write-wins is sufficient | Standard REST semantics are fine. No websockets, no CRDT |
| Automated purchase orders / reorder alerts | Common in inventory software | No stock levels tracked means no reorder point can be computed. There's nothing to trigger an alert on | When a supply price hasn't been updated in 60+ days, a simple "stale price" indicator is sufficient and far simpler |
| Product variants as a single listing | Useful for e-commerce catalogs | Nemea's model uses 5 independent dimensions with explicit SKUs per combination. A "variants" abstraction would fight the existing data model | Keep the explicit 5-dimension model. Each combination is its own product row with its own SKU |
| Tiendanube integration (v1) | The calculadora exists as a prototype | The user explicitly descoped this from v1. The calculadora is self-contained and doesn't need live backend data in v1 | Build the static calculadora in v2 connected to config_tiendanube |

---

## Feature Dependencies

```
Google OAuth (Auth)
    └──required by──> All protected routes
                          └──required by──> Supply CRUD
                          └──required by──> Product CRUD
                          └──required by──> Expense tracking
                          └──required by──> Catalog management

Supply CRUD
    └──required by──> Price history per supply
                          └──required by──> Dynamic cost calculation

Product CRUD
    └──required by──> BOM (material composition)
                          └──required by──> Dynamic cost calculation
                          └──required by──> BOM version history

Catalog management (5 dimension tables)
    └──required by──> Product CRUD
                          (cannot create a product without type, name, finish, color, size options)

Dynamic cost calculation
    └──enhances──> Investor-ready product view
                      (cost + margin = investor value)

Supplier CRUD
    └──required by──> Supply CRUD
                          (every supply has a supplier_id FK)

Supply type CRUD
    └──required by──> Supply CRUD
                          (every supply has a type_id FK)

Selling price history
    └──enhances──> Dynamic cost calculation
                      (cost + selling price = margin)
```

### Dependency Notes

- **Auth requires everything else:** No feature is accessible without the whitelist + role system. Auth must be the first backend module.
- **Catalog management required before Product CRUD:** The 5 dimension tables must have seed data before a product can be created. Pre-populate with the owner's real catalog on first deploy.
- **Supplier CRUD before Supply CRUD:** The FK constraint means suppliers must exist first. Can be seeded.
- **BOM requires both Supply and Product to exist:** It's the join between them. Cannot be created until both parent records exist.
- **Cost calculation is free:** It's not a separate feature to build — it's a query that joins existing tables. The "feature" is the data model being correct.

---

## MVP Definition

### Launch With (v1)

This is the Sheets replacement. All of these must exist or the app has negative value compared to Sheets.

- [ ] Google OAuth authentication with email whitelist and ADMIN/USER roles — without this nothing else is accessible
- [ ] Catalog CRUD (5 product dimensions + supply types + suppliers) — prerequisite for all other CRUDs
- [ ] Supply CRUD with price history — the foundation of cost calculations
- [ ] Product CRUD with 5-dimension model and auto-generated SKU — the core entity
- [ ] BOM management (material composition per product) with version history — required for cost to be calculable
- [ ] Dynamic cost calculation (surfaced in the products list view) — the core value proposition
- [ ] Expense tracking with categories — parity with Sheets
- [ ] Selling price history per product — needed to compute margin
- [ ] Investor read-only view (same product list, restricted role) — named stakeholder requirement
- [ ] Soft delete for products and supplies — data integrity, required from day one

### Add After Validation (v1.x)

These add polish once core workflows are stable.

- [ ] Cost breakdown by material type in product detail — adds analytical value once data is populated
- [ ] "Stale price" indicator on supplies (price not updated in N days) — operational UX improvement
- [ ] Export product list with costs to CSV — investors may want offline copy

### Future Consideration (v2+)

Explicitly out of scope per PROJECT.md. Document here to prevent scope creep.

- [ ] Tiendanube config + calculadora — v2, user descoped it from v1
- [ ] B2B clients and orders — v2
- [ ] Investor dashboard with metrics — v2 (no metrics defined yet)
- [ ] Stock quantity tracking — v3+ (requires orders/fulfillment first)
- [ ] Labor cost in BOM — v3+ (requires time-tracking process)
- [ ] Multi-currency prices — v2+ (only if ARS/USD gap becomes operationally painful)
- [ ] Data migration from Google Sheets — final phase of v1, after app is stable

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Auth (Google OAuth + roles) | HIGH | MEDIUM | P1 |
| Catalog CRUD (5 dimensions) | HIGH | LOW | P1 |
| Supply CRUD + price history | HIGH | LOW | P1 |
| Product CRUD + SKU generation | HIGH | MEDIUM | P1 |
| BOM management + version history | HIGH | MEDIUM | P1 |
| Dynamic cost calculation | HIGH | LOW | P1 — free from architecture |
| Expense tracking | MEDIUM | LOW | P1 — parity with Sheets |
| Selling price history | MEDIUM | LOW | P1 — needed for margin |
| Investor read-only view | MEDIUM | LOW | P1 — named requirement |
| Soft delete everywhere | HIGH | LOW | P1 — data integrity |
| Cost breakdown by supply type | MEDIUM | LOW | P2 |
| Stale price indicator | LOW | LOW | P2 |
| CSV export | LOW | LOW | P2 |
| Tiendanube calculadora | HIGH | HIGH | P3 (v2) |
| B2B orders | HIGH | HIGH | P3 (v2) |

**Priority key:**
- P1: Must have for launch — the Sheets replacement is not complete without these
- P2: Should have — adds polish, implement in same phase if time allows
- P3: Defer — out of scope for v1

---

## Competitor Feature Analysis

Context: Nemea is not building a product to compete commercially. This analysis shows what features comparable tools include, to validate what's table stakes vs what's over-engineering for a 1-admin artisan business.

| Feature | Craftybase (artisan-focused) | Katana MRP (small manufacturer) | Nemea v1 Approach |
|---------|------------------------------|---------------------------------|-------------------|
| Bill of Materials / recipe | Yes, multi-level | Yes, multi-level | Single-level BOM (sufficient for leather goods with no sub-assemblies) |
| Material cost tracking | Yes, weighted average | Yes, FIFO/AVCO | Last-price method — simpler, matches current Sheets logic |
| Price history | Yes (material cost over time) | Yes | Append-only history table with date |
| Supplier management | Yes | Yes | Yes — suppliers table with contact info |
| Labor cost | Yes | Yes | No — deliberate anti-feature for v1 |
| Inventory levels / stock tracking | Yes | Yes | No — deliberate anti-feature for v1 |
| Sales channel integration | Yes (Etsy, Shopify, etc.) | Yes (Shopify) | No — deliberate anti-feature for v1 |
| Role-based access | Limited | Yes | Yes — ADMIN/USER with Google OAuth |
| Expense tracking | Yes | Partial | Yes — categories match Sheets |
| SKU management | Yes | Yes | Yes — auto-generated from 5 dimensions |

**Conclusion:** Nemea's v1 feature set is the lean intersection of what Craftybase offers that actually applies to this business, stripped of everything that doesn't map to current operations (stock levels, labor, sales integrations). This is correct — the goal is Sheets parity with better data integrity and access control, not feature parity with a full MRP.

---

## Sources

- Craftybase feature set: [craftybase.com](https://craftybase.com/) — artisan/maker-focused BOM and cost tracking software (MEDIUM confidence, WebSearch-derived)
- Katana MRP BOM features: [katanamrp.com/bill-of-materials-software](https://katanamrp.com/bill-of-materials-software/) — small manufacturer reference (MEDIUM confidence, WebSearch-derived)
- NetSuite manufacturing inventory guide: [netsuite.com](https://www.netsuite.com/portal/resource/articles/erp/manufacturing-inventory-management.shtml) — manufacturing inventory concepts (MEDIUM confidence)
- Leather manufacturing pain points: [controlata.com/for/leather](https://controlata.com/for/leather/) — leather-specific software (MEDIUM confidence, WebSearch-derived)
- Nemea business rules: `.claude/docs/planning/business-rules.md` — primary source (HIGH confidence, first-party)
- Nemea scope v1: `.claude/docs/planning/scope-v1.md` — primary source (HIGH confidence, first-party)
- Project context: `.planning/PROJECT.md` — primary source (HIGH confidence, first-party)

---

*Feature research for: product cost management / artisan leather goods workshop*
*Researched: 2026-02-28*
