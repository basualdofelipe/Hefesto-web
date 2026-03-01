---
phase: 03-catalogs-and-suppliers
plan: 01
subsystem: database
tags: [uuid, typeorm, migration, postgresql, base-entity]

# Dependency graph
requires:
  - phase: 02-auth
    provides: "Users table with integer PK, auth flow with JWT"
provides:
  - "UUID-based BaseEntity for all future entities"
  - "users.id migrated from integer SERIAL to UUID"
  - "All TypeScript id references use string type"
  - "ParseUUIDPipe for UUID param validation"
affects: [03-catalogs-and-suppliers, 04-supplies-and-price-history, 05-products-and-bom]

# Tech tracking
tech-stack:
  added: [uuid-ossp]
  patterns: [uuid-primary-keys, ParseUUIDPipe-params]

key-files:
  created:
    - nemea-back/src/database/migrations/1772380000000-AlterUsersIdToUuid.ts
  modified:
    - nemea-back/src/common/entities/base.entity.ts
    - nemea-back/src/auth/strategies/jwt.strategy.ts
    - nemea-back/src/auth/decorators/current-user.decorator.ts
    - nemea-back/src/auth/auth.service.ts
    - nemea-back/src/auth/dto/auth-response.dto.ts
    - nemea-back/src/users/users.service.ts
    - nemea-back/src/users/users.controller.ts
    - nemea-front/src/auth.ts

key-decisions:
  - "UUID migration uses add-column/drop-column strategy (not ALTER TYPE) for clean SERIAL-to-UUID conversion"
  - "PK constraint renamed from auto-generated hash to semantic PK_users_id"
  - "Test files updated with consistent mock UUID a1b2c3d4-e5f6-7890-abcd-ef1234567890"

patterns-established:
  - "UUID PKs: All entities inherit from BaseEntity with @PrimaryGeneratedColumn('uuid')"
  - "Route params: Use ParseUUIDPipe (not ParseIntPipe) for :id params"
  - "Mock UUIDs: Use a1b2c3d4-e5f6-7890-abcd-ef1234567890 as standard test mock ID"

requirements-completed: [CATL-01, CATL-02, CATL-03, CATL-04, CATL-05, CATL-06, SUPP-01, SUPP-02]

# Metrics
duration: 7min
completed: 2026-03-01
---

# Phase 3 Plan 1: UUID Migration Summary

**UUID primary keys project-wide: BaseEntity with uuid_generate_v4(), users table migrated from integer SERIAL to UUID, all TypeScript id references updated to string**

## Performance

- **Duration:** 7 min
- **Started:** 2026-03-01T15:49:20Z
- **Completed:** 2026-03-01T15:56:05Z
- **Tasks:** 2
- **Files modified:** 12

## Accomplishments
- Created and ran UUID migration for users table (both dev and test databases)
- Updated BaseEntity to use @PrimaryGeneratedColumn('uuid') with string type
- Updated all backend TypeScript references: JwtPayload, JwtUser, AuthService, UsersService, UsersController, AuthResponseDto
- Replaced ParseIntPipe with ParseUUIDPipe in route params
- Updated frontend BackendAuthResponse to use string id
- Fixed all spec files to use UUID mock values (31 tests passing)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create UUID migration for users table** - `ab88eca` (feat) [nemea-back]
2. **Task 2: Update BaseEntity to UUID and fix all id type references** - `edee3cc` (feat) [nemea-back] + `14cebc7` (feat) [nemea-front]

## Files Created/Modified
- `nemea-back/src/database/migrations/1772380000000-AlterUsersIdToUuid.ts` - UUID migration: enable uuid-ossp, swap integer PK for UUID PK
- `nemea-back/src/common/entities/base.entity.ts` - @PrimaryGeneratedColumn('uuid'), id: string
- `nemea-back/src/auth/strategies/jwt.strategy.ts` - JwtPayload.sub: string, validate() returns id: string
- `nemea-back/src/auth/decorators/current-user.decorator.ts` - JwtUser.id: string
- `nemea-back/src/auth/auth.service.ts` - getProfile(userId: string)
- `nemea-back/src/auth/dto/auth-response.dto.ts` - AuthUserDto.id: string with UUID Swagger example
- `nemea-back/src/users/users.service.ts` - All methods: id params changed to string
- `nemea-back/src/users/users.controller.ts` - ParseUUIDPipe replaces ParseIntPipe
- `nemea-front/src/auth.ts` - BackendAuthResponse user.id: string
- `nemea-back/src/auth/auth.controller.spec.ts` - Mock IDs updated to UUID strings
- `nemea-back/src/auth/auth.service.spec.ts` - Mock IDs and assertions updated to UUIDs
- `nemea-back/src/users/users.controller.spec.ts` - Mock IDs updated to UUID strings
- `nemea-back/src/users/users.service.spec.ts` - Mock IDs and all method call assertions updated to UUIDs

## Decisions Made
- UUID migration uses add-column/drop-column strategy instead of ALTER TYPE for clean SERIAL-to-UUID conversion (PostgreSQL cannot ALTER COLUMN TYPE from integer to uuid directly)
- PK constraint renamed from auto-generated hash (PK_a3ffb1c0c8416b9fc6f907b7433) to semantic name (PK_users_id) for readability
- Used consistent mock UUID (a1b2c3d4-e5f6-7890-abcd-ef1234567890) across all test files for predictability

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed all spec files with number-typed mock IDs**
- **Found during:** Task 2 (TypeScript compilation check)
- **Issue:** 4 spec files used `id: 1` or numeric arguments for service/controller methods that now expect string UUIDs. TypeScript compilation failed with 17 type errors.
- **Fix:** Updated all mock IDs and method call arguments in auth.controller.spec.ts, auth.service.spec.ts, users.controller.spec.ts, and users.service.spec.ts to use UUID strings.
- **Files modified:** 4 spec files
- **Verification:** `npx tsc --noEmit` passes, `npx jest` 31/31 tests pass
- **Committed in:** edee3cc (part of Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug fix)
**Impact on plan:** Test files were a direct consequence of the type changes. No scope creep.

## Issues Encountered
- Pre-commit hook failed on first migration commit due to prettier formatting -- ran `prettier --write` to fix, second commit succeeded.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- BaseEntity now uses UUID for all future entities
- Ready for Plan 02 (catalog dimension entities) which will extend BaseEntity
- Ready for Plan 03 (supplier entity) which will extend BaseEntity
- No blockers for remaining Phase 3 plans

## Self-Check: PASSED

- FOUND: nemea-back/src/database/migrations/1772380000000-AlterUsersIdToUuid.ts
- FOUND: nemea-back/src/common/entities/base.entity.ts
- FOUND: .planning/phases/03-catalogs-and-suppliers/03-01-SUMMARY.md
- FOUND: commit ab88eca (Task 1 migration)
- FOUND: commit edee3cc (Task 2 backend types)
- FOUND: commit 14cebc7 (Task 2 frontend types)

---
*Phase: 03-catalogs-and-suppliers*
*Completed: 2026-03-01*
