---
phase: 01-backend-scaffold
plan: 02
subsystem: database
tags: [docker, postgresql, typeorm, docker-compose, migrations, base-entity]

# Dependency graph
requires:
  - phase: 01-backend-scaffold plan 01
    provides: NestJS 11 skeleton with TypeORM/pg dependencies installed
provides:
  - Docker Compose with PostgreSQL 16 dev (5432) and test (5433) databases
  - Shared TypeORM DataSource config (single source of truth for CLI and NestJS runtime)
  - Windows-compatible migration scripts (no $npm_config_name)
  - Abstract BaseEntity with id, created_at, updated_at timestamps
  - Role enum (ADMIN, USER) for Phase 2 auth
  - E2E test infrastructure targeting test database
affects: [01-03-health-swagger, 02-auth, 03-catalogs, 04-supplies]

# Tech tracking
tech-stack:
  added: ["dotenv/config (via typeorm)", "postgres:16-alpine (Docker)"]
  patterns: [shared-datasource, docker-compose-dev-test, base-entity-inheritance, e2e-test-database-isolation]

key-files:
  created: ["nemea-back/docker-compose.yml", "nemea-back/.env.example", "nemea-back/src/database/data-source.ts", "nemea-back/src/config/typeorm.config.ts", "nemea-back/src/database/migrations/.gitkeep", "nemea-back/src/common/entities/base.entity.ts", "nemea-back/src/common/types/role.enum.ts", "nemea-back/test/setup-e2e.ts"]
  modified: ["nemea-back/package.json", "nemea-back/src/app.module.ts", "nemea-back/test/jest-e2e.json"]

key-decisions:
  - "Shared DataSource config pattern: data-source.ts exports dataSourceOptions used by both TypeORM CLI and NestJS TypeOrmModule"
  - "synchronize: false unconditional -- never conditioned on NODE_ENV, migrations are the only schema change mechanism"
  - "migrationsRun: true -- migrations auto-execute on app startup for simplified deployment"
  - "Definite assignment assertion (!) on BaseEntity properties for TypeScript strict compatibility with TypeORM decorators"
  - "E2E test isolation via setup-e2e.ts overriding DATABASE_URL to point to separate test database on port 5433"

patterns-established:
  - "Shared DataSource pattern: src/database/data-source.ts is the single source of truth, src/config/typeorm.config.ts spreads it for NestJS"
  - "Docker Compose dev/test pattern: dev DB on 5432, test DB on 5433, both postgres:16-alpine with health checks"
  - "BaseEntity inheritance pattern: all future entities extend BaseEntity for id + timestamps"
  - "Migration script pattern: npm run migration:generate -- src/database/migrations/MigrationName (Windows-safe)"

requirements-completed: [INFR-01, INFR-02, INFR-03, INFR-04]

# Metrics
duration: 5min
completed: 2026-02-28
---

# Phase 1 Plan 02: Docker + TypeORM + BaseEntity Summary

**Docker Compose with dual PostgreSQL databases, shared TypeORM DataSource config (CLI + runtime), abstract BaseEntity with timestamps, and Windows-compatible migration scripts**

## Performance

- **Duration:** 5 min
- **Started:** 2026-02-28T20:26:13Z
- **Completed:** 2026-02-28T20:31:07Z
- **Tasks:** 2
- **Files modified:** 11

## Accomplishments
- Docker Compose runs PostgreSQL 16 with health checks for both dev (5432) and test (5433) databases
- TypeORM DataSource is a single source of truth -- same config for CLI migrations and NestJS runtime, preventing config drift
- NestJS app boots and connects to Docker PostgreSQL, auto-runs migrations on startup
- BaseEntity establishes the timestamp convention (created_at, updated_at as timestamptz) for all future entities
- Migration scripts are Windows-compatible (no $npm_config_name -- user passes path as trailing argument)

## Task Commits

Each task was committed atomically:

1. **Task 1: Docker Compose, .env files, and shared DataSource configuration** - `0fb7acc` (feat)
2. **Task 2: BaseEntity with timestamps and wire TypeORM into AppModule** - `3314db5` (feat)

Note: Both commits are in the nemea-back/ sub-repo on the `development` branch.

## Files Created/Modified
- `nemea-back/docker-compose.yml` - PostgreSQL 16 dev + test services with health checks and volume
- `nemea-back/.env.example` - Complete env template with all dev values pre-filled
- `nemea-back/src/database/data-source.ts` - Single source of truth for TypeORM DataSource config
- `nemea-back/src/config/typeorm.config.ts` - NestJS TypeOrmModule config spreading shared dataSourceOptions
- `nemea-back/src/database/migrations/.gitkeep` - Empty migrations directory for future phases
- `nemea-back/src/common/entities/base.entity.ts` - Abstract base entity with id, created_at, updated_at
- `nemea-back/src/common/types/role.enum.ts` - Role enum (ADMIN, USER) for Phase 2 auth
- `nemea-back/src/app.module.ts` - Updated to import TypeOrmModule.forRoot(typeOrmConfig)
- `nemea-back/package.json` - Added typeorm, migration:generate, migration:run, migration:revert scripts
- `nemea-back/test/jest-e2e.json` - Updated with path aliases and setup file
- `nemea-back/test/setup-e2e.ts` - E2E setup overriding DATABASE_URL to test database

## Decisions Made
- Shared DataSource pattern: data-source.ts is the single source of truth, typeorm.config.ts spreads it -- prevents CLI/runtime config drift (the #1 TypeORM pitfall)
- synchronize: false is unconditional (not conditioned on NODE_ENV) -- migrations are the only schema change mechanism
- migrationsRun: true for auto-execution on app start -- simplifies Railway deployment
- Used definite assignment assertion (!) on BaseEntity properties -- required by TypeScript strict mode when decorators initialize properties
- E2E test isolation via setup-e2e.ts overriding DATABASE_URL -- keeps test data separate from dev data

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Added definite assignment assertion to BaseEntity properties**
- **Found during:** Task 2 (BaseEntity creation)
- **Issue:** TypeScript strict mode (strictPropertyInitialization) requires properties to be initialized in constructor or marked with `!`. TypeORM decorators initialize properties at runtime, not at construction.
- **Fix:** Added `!` definite assignment assertion to id, createdAt, and updatedAt properties
- **Files modified:** nemea-back/src/common/entities/base.entity.ts
- **Verification:** `npm run build` passes with 0 errors
- **Committed in:** 3314db5

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Standard TypeORM + strict TS pattern. No scope creep.

## Issues Encountered
- Port 5433 was occupied by an existing postgres-cuerpo-fit container -- stopped it temporarily during verification, restarted it after Docker Compose down

## User Setup Required

None - no external service configuration required. Copy .env.example to .env (already done) and run `docker compose up -d` to start databases.

## Next Phase Readiness
- Database foundation complete for Plan 03 (health check, Swagger, global config)
- All future entities will extend BaseEntity for consistent timestamps
- Migration workflow ready: `npm run migration:generate -- src/database/migrations/CreateTableName`
- E2E test infrastructure ready for Phase 2+ integration tests

## Self-Check: PASSED

All 8 created files verified on disk. Both commit hashes (0fb7acc, 3314db5) verified in git log. SUMMARY.md exists.

---
*Phase: 01-backend-scaffold*
*Completed: 2026-02-28*
