# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-28)

**Core value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automáticamente cuando cambian los precios de los insumos.
**Current focus:** Phase 1 — Backend Scaffold

## Current Position

Phase: 1 of 7 (Backend Scaffold)
Plan: 0 of 3 in current phase
Status: Ready to plan
Last activity: 2026-02-28 — Roadmap created from requirements + research

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: -
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**
- Last 5 plans: none yet
- Trend: -

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Roadmap: synchronize: false hardcoded unconditionally — never conditioned on NODE_ENV (Pitfall 1)
- Roadmap: Two TypeORM DataSource configs share the same entities array from src/database/dataSource.ts (Pitfall 2)
- Roadmap: CostsModule is separate from ProductsModule to avoid circular deps (imports both SuppliesModule + ProductsModule)
- Roadmap: TiendanubeConfigModule named explicitly to avoid collision with @nestjs/config's ConfigModule
- Roadmap: Soft delete strategy (is_active vs @DeleteDateColumn) — decision deferred to Phase 3/4 planning

### Pending Todos

None yet.

### Blockers/Concerns

- Phase 2 (Auth): NextAuth jwt/session callback + NestJS POST /auth/google token exchange is the highest-complexity item — must be verified end-to-end before building on top of it. See .planning/research/PITFALLS.md.
- Phase 4 (Supplies): Soft delete + unique constraint strategy must be decided before the first entity with unique constraints is migrated. Options: is_active partial index vs @DeleteDateColumn. Decide during Phase 3 planning.
- Phase 6 (Costs): DISTINCT ON batched query must be verified against NestJS logger to confirm exactly 2 SQL queries for the product list (explicit acceptance criterion).

## Session Continuity

Last session: 2026-02-28
Stopped at: Roadmap created. ROADMAP.md, STATE.md, and REQUIREMENTS.md traceability written. Ready to plan Phase 1.
Resume file: None
