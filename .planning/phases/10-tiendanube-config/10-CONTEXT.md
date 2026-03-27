---
phase: 10
name: Tiendanube Config
discussed: 2026-03-27
---

# Phase 10: Tiendanube Config — Context

## Domain

Admin-editable configuration tables for Tiendanube payment gateway rates, installment fees, and tax settings. These tables feed the calculadora (Phase 11) and investor dashboard (Phase 13) as single source of truth for all pricing calculations.

## Key Discovery

The original prototype only modeled Pago Nube with 4 Tiendanube plans. The real system has **3 payment gateways** (Pago Nube, Mercado Pago, MODO) each with their own rate structures. This is significantly more complex than initially scoped.

## Decisions

### Schema: 3+ tables, normalized by concept

**Decision:** Separate tables for distinct data domains. Not a single JSON blob or key-value store.

Tables needed:
1. **tn_payment_gateways** — Gateway master (Pago Nube, Mercado Pago, MODO) with name, is_active, status
2. **tn_gateway_rates** — Rates per gateway: payment_method, withdrawal_days, rate_percent. Append-only with history (same pattern as supply prices).
3. **tn_installment_rates** — Installment tiers: installments (1,3,6,9,12), rate_percent. Append-only with history.
4. **tn_tax_config** — Tax settings: IVA rate, IIBB alicuota. Append-only with history.
5. **tn_plans** — Tiendanube plan (Inicial, Esencial, Impulso, Escala) with CPT rates per gateway type (Pago Nube always 0%, others vary by plan).

**Why:** Professional, portfolio-quality. Each concept is its own entity. Supports N gateways, N payment methods, N withdrawal timings without schema changes.

### Multiple Payment Gateways

**Decision:** Model all 3 gateways from day 1.

| Gateway | Status | Rate Structure |
|---------|--------|---------------|
| **Pago Nube** | Activado | Tarjeta déb/créd (7d, 14d), Billetera Virtual (1d, 7d, 14d), Transferencia (1d). CPT = gratis. |
| **Mercado Pago** | Activado | Todos los medios: al momento, 10d, 18d, 35d. CPT = 2% (plan Esencial). |
| **MODO** | Pendiente | Tarjeta crédito (1d, 8d), Tarjeta débito (1d). CPT = 2% (plan Esencial). |

Each gateway has different payment methods and different withdrawal timing options. The schema must be flexible enough to handle this heterogeneity.

### CPT (Costo Por Transacción de Tiendanube)

**Decision:** CPT depends on the Tiendanube plan AND the gateway:
- Pago Nube: always 0% (bonificado en todos los planes)
- Other gateways: depends on plan — Inicial: N/A (solo Pago Nube), Esencial: 2%, Impulso: 1%, Escala: 0.7%

The admin selects their current Tiendanube plan, and the CPT is derived automatically.

### Rate History

**Decision:** Append-only history (same pattern as supply price history). When admin changes a rate, a new record is created. The latest record is the active one. Previous values are preserved.

**Why:** User explicitly chose this. Allows tracking how gateway commissions evolved over time. Consistent with the existing supplies_price_history pattern.

### UI: Page with collapsible sections

**Decision:** Dedicated page at `/configuracion/tiendanube` with:
- Plan Tiendanube dropdown at the top (Esencial, Impulso, Escala)
- Collapsible section per gateway (Pago Nube, Mercado Pago, MODO)
  - Table inside with: Medio de pago, Tiempo de retiro, Tasa (editable), CPT (derived)
  - Guardar button per gateway section
- Collapsible section for Cuotas (installment rates)
- Collapsible section for Impuestos (IVA, IIBB)
- "Verificar tasas" link at the bottom → official Pago Nube rates page

Sidebar: new link under "Admin" section → "Config Tiendanube"

### Seed Data

Seed with verified March 2026 rates from screenshots:

**Pago Nube (Plan Esencial):**
- Tarjeta déb/créd: 7d=4.39%, 14d=3.49%
- Billetera Virtual: 1d=6.09%, 7d=4.39%, 14d=3.49%
- Transferencia: 1d=1.50%

**Mercado Pago:**
- Todos los medios: 0d=6.29%, 10d=4.39%, 18d=3.39%, 35d=1.49%

**MODO:**
- Tarjeta crédito: 1d=7.11%, 8d=2.80%
- Tarjeta débito: 1d=1.80%

**Cuotas:** 1=0%, 3=8.42%, 6=17.41%, 9=27.04%, 12=37.39%
**Impuestos:** IVA=21%, IIBB=3.5% (configurable)
**Plan:** Esencial (CPT: Pago Nube=0%, otros=2%)

## Specifics

- All rates are "% + IVA" — the rate stored in DB is the base rate, IVA is calculated on top
- IIBB alicuota varies per vendor (SIRTAC/COMARB). Not automatable. Admin enters their rate manually.
- Plan "Inicial" only allows Pago Nube (no other gateways). "Evolución" plan exists but is "Negociable" — ignore for now.
- MODO is "Pendiente" (not yet activated) — seed the data but mark gateway as inactive.

## Deferred Ideas

- Auto-fetch rates from Tiendanube (no API available for this)
- Side-by-side plan comparison ("what if I upgrade to Impulso?")
- Rate change notifications

## Canonical Refs

- `_archive/calculadora/calculadora-tiendanube.jsx` — Prototype with DEFAULT_PLANS, DEFAULT_CUOTAS_TASAS
- `.claude/docs/planning/business-rules.md` — calcForward 14-step formula
- `.planning/research/FEATURES.md` — Feature landscape with TN config as table stake
- `.planning/research/ARCHITECTURE.md` — TiendanubeConfigModule design
- `.planning/research/PITFALLS.md` — Pitfall 7 (stale rates), Pitfall 10 (decimal string convention)

## Claude's Discretion

- Exact entity field names and TypeORM decorators
- Migration timestamp and naming
- DTO validation rules (min/max for rate percentages)
- How to structure the NestJS module (TiendanubeConfigModule naming to avoid collision with @nestjs/config)
- Whether to use separate controllers per table or one unified controller
- Collapsible component choice (Radix Collapsible vs Accordion)
