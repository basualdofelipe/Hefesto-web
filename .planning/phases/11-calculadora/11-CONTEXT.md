---
phase: 11
name: Calculadora
discussed: 2026-03-27
---

# Phase 11: Calculadora — Context

## Domain

Forward (precio→ganancia) and inverse (ganancia→precio) Tiendanube pricing simulation using real product costs from DB + admin-editable gateway rates from Phase 10's TiendanubeConfigModule.

## Decisions

### Multi-gateway support

**Decision:** The calculadora simulates sales through ANY of the 3 payment gateways (Pago Nube, Mercado Pago, MODO). The user selects:
1. Producto (auto-fills cost from DB)
2. Pasarela (Pago Nube / MP / MODO — only active gateways shown)
3. Medio de pago (depends on selected gateway — e.g., tarjeta, billetera, transferencia)
4. Tiempo de retiro (depends on selected gateway + payment method)
5. Cuotas (1, 3, 6, 9, 12)

Selectors are cascading: changing pasarela updates medio options, changing medio updates retiro options.

### Plan TN: inherit + override

**Decision:** The calculadora inherits the admin's current TN plan from the config. A toggle "Simular otro plan" reveals a plan selector to override. Default: plan from config. This avoids a 6th selector by default.

### calcForward adaptation for multiple gateways

**Decision:** The prototipo's calcForward assumes Pago Nube structure. The new version must:
- Accept `gatewayRate` (commission %) instead of looking up from a plan object
- Accept `cpt` (% from TnPlan based on gateway type) instead of `txTiendanube`
- Accept `installmentRate` (% from TnInstallmentRate)
- Accept `ivaRate` and `iibbRate` (% from TnTaxConfig)

The 14-step formula stays the same conceptually but pulls rates from the resolved gateway/method/withdrawal combination instead of hardcoded plan objects.

### UI: Two-column layout

**Decision:** Page at `/calculadora` with:
- Left column: mode toggle (forward/inverse), product selector (auto-fills cost), price/profit input, shipping cost input, gateway selectors
- Right column: full desglose (total cliente, comisión pasarela, financiación cuotas, IVA neto, IIBB, CPT, neto recibido, costo producto con IVA, GANANCIA REAL, margen %)
- Both ADMIN and USER can access (it's a simulation tool, read-only data)

### Inverse mode edge cases

**Decision:** Validate inputs and show clear error messages:
- Cost = $0 or null → "Definí el costo del producto primero"
- Desired profit negative → "La ganancia deseada debe ser positiva"
- Desired profit unreachable (binary search converges to absurd price) → "Ganancia inalcanzable con estas tasas"
- Binary search convergence: epsilon = $0.01, max 100 iterations (same as prototype)
- Upper bound: costoProducto × 20 (same as prototype)

### Backend: pure CalculadoraService

**Decision (from research):** CalculadoraService is a stateless service that:
- Imports TiendanubeConfigModule (for rates) and CostsModule (for product costs)
- Has `calcForward()` and `calcInverse()` methods
- Has `calcBatch()` for Phase 13 dashboard
- Does NOT write to any table — pure calculation
- Rounds only once at the end of the full chain (not at intermediate steps)

Endpoints:
- `POST /calculadora/forward` — price → profit
- `POST /calculadora/inverse` — profit → price
- `POST /calculadora/batch` — all products margins (Phase 13 dependency)

### Tests (from TODO)

**Decision:** CalculadoraService MUST have unit tests:
- Test calcForward with Billetera Hefesto at $87,000 (known values from prototype)
- Test calcInverse round-trip: forward(price) → profit, inverse(profit) → price, verify within $0.01
- Test edge cases: zero-cost product, negative target profit, unreachable target

## Specifics

- Product selector shows all active products grouped by type→name (Phase 9 hierarchy)
- When product selected, cost auto-fills from CostsService.calculateForProduct()
- Envío (shipping cost) is a manual input (not from DB — user enters per-sale)
- In inverse mode, the "Precio" field becomes the OUTPUT (calculated), and a "Ganancia deseada" field appears as INPUT
- The desglose shows all 14 intermediate values so the user can verify each step

## Folded Todos

- "Include CalculadoraService tests in Phase 11 plan" — tests with Hefesto $87,000 round-trip

## Deferred Ideas

- Compare results across gateways side-by-side ("¿cuál me conviene más?")
- Historical simulation ("¿cuánto hubiera ganado con las tasas de hace 3 meses?")
- Bulk calculadora for multiple products at once (partially covered by batch endpoint)

## Canonical Refs

- `_archive/calculadora/calculadora-tiendanube.jsx` — Prototype with calcForward/calcInverse
- `.claude/docs/planning/business-rules.md` — calcForward 14-step formula documentation
- `.planning/phases/10-tiendanube-config/10-CONTEXT.md` — TN config schema decisions
- `.planning/phases/10-tiendanube-config/10-01-SUMMARY.md` — Backend API contract
- `.planning/research/PITFALLS.md` — Pitfall 1 (floating point), Pitfall 6 (inverse edge cases)
- `.planning/todos/pending/2026-03-26-calculadora-service-tests-round-trip.md` — Test requirements

## Claude's Discretion

- Exact DTO field names for forward/inverse requests
- How to resolve the "current rate" for a gateway+method+withdrawal combination (DISTINCT ON or service method)
- Cascading selector implementation (separate API calls vs single getAll + filter client-side)
- Shadcn component choices for the calculator UI (Card, Input, Select, RadioGroup, etc.)
- Whether to debounce the calculation or compute on every input change
