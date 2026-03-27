---
gsd_state_version: 1.0
milestone: v1.1
milestone_name: tiendanube-investor-dashboard
status: executing
last_updated: "2026-03-27"
progress:
  total_phases: 6
  completed_phases: 3
  total_plans: 5
  completed_plans: 5
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-26)

**Core value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automaticamente cuando cambian los precios de los insumos.
**Current focus:** Milestone v1.1 — Phase 10 complete (Tiendanube Config). Next: Phase 11 (Calculadora).

## Current Position

Phase: 10 of 13 (Tiendanube Config) — COMPLETE
Plan: 2 of 2 (all complete)
Status: Phase 10 done. Both backend (10-01) and frontend (10-02) plans executed.
Last activity: 2026-03-27 — Phase 10 completed (10-02 frontend config page)

Progress (v1.1): [############............] 50% (3/6 phases)
Progress (overall): [#######################...] 87% (10/13 phases)

## Performance Metrics

**Velocity (from v1.0):**
- Total plans completed: 21
- Average duration: 6.4min
- Total execution time: ~2.2 hours

**By Phase (v1.0):**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1 - Backend Scaffold | 3 | 20min | 6.7min |
| 2 - Auth | 3 | 24min | 8min |
| 3 - Catalogs & Suppliers | 4 | 40min | 10min |
| 4 - Supplies & Price History | 3 | 27min | 9min |
| 5 - Products & BOM | 4 | 31min | 7.8min |
| 6 - Cost Calculation | 2 | 11min | 5.5min |
| 7 - Expenses | 2 | ~8min | ~4min |

**v1.1 Metrics:**

| Phase | Plan | Duration | Tasks | Files |
|-------|------|----------|-------|-------|
| 8 - Hardening | 08-01 | 5min | 3 | 8 |
| 8 - Hardening | 08-02 | 9min | 2 | 22 |
| 9 - Product UX | 09-01 | 6min | 2 | 4 |
| 10 - TN Config | 10-01 | 4min | 2 | 14 |
| 10 - TN Config | 10-02 | 7min | 2 | 10 |

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: TiendanubeConfigModule named explicitly to avoid collision with @nestjs/config's ConfigModule
- Roadmap: CostsModule is separate from ProductsModule to avoid circular deps
- v1.1: Hardening must complete before investors access the system (role enforcement gap)
- v1.1: CalculadoraService is pure backend — forward formula never duplicated in frontend
- v1.1: Scenarios use separate scenario_overrides table, never write to product_price_history
- v1.1: Zero new npm dependencies — existing stack covers all v1.1 features
- v1.1: Round financial calculations once at chain end, not at intermediate steps
- Phase 8: proxy.ts renamed to middleware.ts (export name change to `middleware`) — proxy.ts was NOT firing as Next.js middleware
- Phase 8: @Roles(Role.ADMIN) already on all mutation endpoints — HARD-03 pre-satisfied, verify only
- Phase 8: UsersController hard DELETE replaced with PATCH toggle-status (soft delete pattern)
- Phase 8: deactivateBySupplier dead code deleted (never called)
- Phase 8: SupplyOption shared type includes optional `supplier?: { name: string }` field for BOM editors
- Phase 8 (revision): 08-02 depends on 08-01 (users page calls toggle-status endpoint created in 08-01)
- Phase 8 (revision): formatDate MUST include { day: '2-digit', month: '2-digit', year: 'numeric' } options
- Phase 8 (revision): cleanSupplierData uses SupplierFormData type (not Record<string, string>)
- Phase 8 (revision): Do NOT create shared formatAmount — collision with expenses/types.ts version
- Phase 8 (revision): SupplyCombobox does not use Check/cn imports — omit from extraction
- Phase 9: BOM group editor moves from type-level to name-level (PRUX-02)
- Phase 9: Batch price button at BOTH type-level AND name-level (type = broad "all Cinturones", name = targeted "all Hercules")
- Phase 9: BOM override detection uses cost-heuristic first, then accurate detection after dialog opens
- Phase 9 (pre-mortem fix): SKU column in ProductFinishGroup is a Link to /productos/[id] — relocated from removed Nombre column
- Phase 9 (pre-mortem fix): onDivergenceDetected callback uses local `divergent` variable, not state (React batching safety)
- Phase 9 (pre-mortem fix): ProductExpandedRow colSpan explicitly 6 (matching 6-column leaf table)
- Phase 10: 5 DB tables for TN config (tn_payment_gateways, tn_gateway_rates, tn_installment_rates, tn_tax_config, tn_plans)
- Phase 10: 3 gateways modeled from day 1 (Pago Nube, Mercado Pago, MODO) — MODO seeded as inactive
- Phase 10: Append-only rate history for gateway rates, installment rates, and tax config (same pattern as supply prices)
- Phase 10: CPT depends on plan + gateway — Pago Nube always 0%, others vary by plan
- Phase 10: All rates are "% + IVA" — stored rate is base, IVA calculated on top by calculadora
- Phase 10: IIBB/SIRTAC = single configurable value (3.5%), admin enters manually
- Phase 10: Frontend config page uses Shadcn Accordion (type="multiple", all expanded by default) for collapsible sections
- Phase 10: Plan selector dropdown defaults to 'esencial' — state lifted to TiendanubeConfigClient
- Phase 10: GatewaySection uses compound component pattern (Header as static property for accordion trigger)
- Phase 10: /configuracion added to ADMIN_ONLY_ROUTES for middleware-level protection

### Pending Todos

10 todos pending. See `.planning/todos/pending/`.
Latest: Fix JWT expiry silent failure - redirect to login on 401 (auth)

Todos absorbed into Phase 8 plans:
- proxy.ts runtime verification → 08-01 Task 1 (renamed to middleware.ts)
- ExpenseFormDialog category Select bug → 08-02 Task 2 (fix with watch())
- Delete deactivateBySupplier dead code → 08-01 Task 2
- Role-based redirect on login → 08-01 Task 1 (USER redirected to / by middleware)

### Blockers/Concerns

- proxy.ts vs middleware.ts naming RESOLVED: proxy.ts export `proxy` is NOT what Next.js 16 needs. The file must be named middleware.ts with export named `middleware`. Previous research was partially incorrect — runtime verification confirmed it was not firing.
- deactivateBySupplier RESOLVED: deleted as dead code in 08-01
- SIRTAC/IIBB aliquot: single configurable value, admin enters manually. Not automated. (Phase 10 — addressed in plan)

### Known Issues

- Auth ClientFetchError on frontend homepage: NextAuth session endpoint returns HTML instead of JSON. Pre-existing from v1.0.

## Session Continuity

Last session: 2026-03-27
Stopped at: Completed 10-02-PLAN.md (Tiendanube Config Frontend)
Resume file: None — next step is Phase 11 (Calculadora) planning or execution
