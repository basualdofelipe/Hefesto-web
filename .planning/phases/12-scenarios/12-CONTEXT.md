---
phase: 12
name: Scenarios
discussed: 2026-03-28
---

# Phase 12: Scenarios — Context

## Domain

Investor-facing what-if scenario simulator. Create named scenarios with overridden selling prices (+ pasarela/plan config), see recalculated margins using real costs + simulated conditions. Scenarios persist in DB, user-scoped.

## Decisions

### Override scope: price + pasarela + plan (full simulation)

**Decision:** Investors can override:
- **Selling price** per product (individual or bulk)
- **Pasarela** for the scenario (e.g., "what if I sell everything via Mercado Pago?")
- **Plan Tiendanube** for the scenario (e.g., "what if I upgrade to Impulso?")

Costs stay real (from DB) — the investor simulates selling conditions, not production costs.

### Data model

**Decision:** 2 new tables:
1. **scenarios** — name, user_id (FK to users), gateway_slug (override), plan_id (override, FK to tn_plans), created_at, updated_at
2. **scenario_overrides** — scenario_id (FK), product_id (FK), override_price (decimal), created_at

**Isolation rule:** scenario_overrides NEVER writes to product_price_history. Completely separate tables. If a scenario is deleted, its overrides cascade-delete.

### User-scoped visibility

**Decision:** Each user sees only their own scenarios. Backend filters by `userId` from JWT. Querying another user's scenario returns 404.

### Bulk override: percentage by type + individual edit

**Decision:** Two mechanisms:
1. **Bulk action:** Select product type (or "Todos") + percentage → applies to all products of that type. E.g., "+10% Cinturones" changes all cinturón override prices to real price × 1.10
2. **Individual edit:** Each product row has an editable price field. User can manually set any override price.

Bulk adjustments apply ON TOP of current overrides (not on top of real prices, unless no override exists yet).

### UI: Dedicated page + dashboard integration

**Decision:**
- **`/escenarios` page** — List of saved scenarios with create/edit/delete. Clicking edit opens the scenario editor with the full product table, bulk actions, and pasarela/plan selectors.
- **Dashboard integration (Phase 13)** — Dashboard has a scenario selector dropdown. Selecting a scenario replaces real selling prices with override prices and recalculates margins. "Sin escenario" shows real data.

Sidebar: link under "Herramientas" (same group as Calculadora).

### Margin calculation in scenarios

**Decision:** For each product in a scenario:
- If override price exists → use override price as precioVenta
- If no override → use real selling price (currentPrice from product_price_history)
- Pass precioVenta + real cost + scenario's pasarela/plan config to CalculadoraService.calcForward()
- Show: real price, override price, real margin, simulated margin, difference

## Specifics

- Scenarios accessible to both ADMIN and USER roles (it's a simulation tool)
- `/escenarios` should NOT be in ADMIN_ONLY_ROUTES
- Maximum scenarios per user: no limit for now (2-3 users, not a concern)
- Scenario name is required, must be unique per user
- When a product is deactivated, its override in a scenario is preserved but marked as "(inactivo)" in the UI
- Bulk override rounds to nearest integer (ARS don't have meaningful centavos at these price points)

## Deferred Ideas

- Export scenario to PDF/Excel
- Side-by-side scenario comparison
- Scenario duplication ("fork from existing")
- Cost overrides (simulate supply price changes) — user explicitly excluded this for now

## Canonical Refs

- `.planning/phases/11-calculadora/11-CONTEXT.md` — CalculadoraService API contract
- `.planning/phases/11-calculadora/11-01-SUMMARY.md` — calcBatch endpoint
- `.planning/research/ARCHITECTURE.md` — ScenariosModule design
- `.planning/research/PITFALLS.md` — Pitfall 4 (scenario data isolation)
- `.planning/research/FEATURES.md` — Scenario simulator feature analysis

## Claude's Discretion

- Exact entity field names and TypeORM decorators
- DTO validation rules
- How to structure the scenario editor UI (inline table vs modal vs full page)
- Whether bulk override is a dialog/modal or inline controls
- How to display the "difference" between real and simulated margins (color coding, arrows, etc.)
- API endpoint design (REST routes for scenarios + overrides + calculate)
