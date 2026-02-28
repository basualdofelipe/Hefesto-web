# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-28)

**Core value:** Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automáticamente cuando cambian los precios de los insumos.
**Current focus:** Phase 1 — Backend Scaffold

## Current Position

Phase: 1 of 7 (Backend Scaffold)
Plan: 1 of 3 in current phase
Status: Executing phase 1
Last activity: 2026-02-28 — Plan 01-01 completed (NestJS scaffold)

Progress: [#░░░░░░░░░] 5%

## Performance Metrics

**Velocity:**
- Total plans completed: 1
- Average duration: 10min
- Total execution time: 0.17 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1 - Backend Scaffold | 1 | 10min | 10min |

**Recent Trend:**
- Last 5 plans: 01-01 (10min)
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
- 01-01: SWC builder with typeCheck: true for fast compilation + type safety
- 01-01: ESLint flat config (not legacy .eslintrc) matching frontend sonarjs/explicit-return-type conventions
- 01-01: Port 4000 in main.ts per CLAUDE.md backend convention
- 01-01: ES2021 target in tsconfig for wider runtime compatibility (scaffold defaulted to ES2023)

### Pending Todos

None yet.

### Blockers/Concerns

- Phase 2 (Auth): NextAuth jwt/session callback + NestJS POST /auth/google token exchange is the highest-complexity item — must be verified end-to-end before building on top of it. See .planning/research/PITFALLS.md.
- Phase 4 (Supplies): Soft delete + unique constraint strategy must be decided before the first entity with unique constraints is migrated. Options: is_active partial index vs @DeleteDateColumn. Decide during Phase 3 planning.
- Phase 6 (Costs): DISTINCT ON batched query must be verified against NestJS logger to confirm exactly 2 SQL queries for the product list (explicit acceptance criterion).

## Session Continuity

Last session: 2026-02-28
Stopped at: Completed 01-01-PLAN.md (NestJS project scaffold)
Resume file: .planning/phases/01-backend-scaffold/01-01-SUMMARY.md
