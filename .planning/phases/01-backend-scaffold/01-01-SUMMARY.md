---
phase: 01-backend-scaffold
plan: 01
subsystem: infra
tags: [nestjs, typescript, swc, eslint, prettier, husky, lint-staged]

# Dependency graph
requires:
  - phase: none
    provides: first backend plan, no prior dependencies
provides:
  - NestJS 11 project skeleton with SWC builder in nemea-back/
  - TypeScript strict mode with path aliases (@src/*)
  - ESLint flat config matching frontend conventions (sonarjs, no-any, explicit-return-type)
  - Prettier config consistent with frontend (single quotes, trailing commas)
  - Husky + lint-staged pre-commit hooks
  - All core dependencies installed (TypeORM, pg, config, validation, helmet, throttler, swagger)
affects: [01-02-docker-env, 01-03-health-swagger, 02-auth]

# Tech tracking
tech-stack:
  added: ["@nestjs/core@11", "@nestjs/typeorm@11", "typeorm", "pg", "@nestjs/config", "joi", "class-validator", "class-transformer", "helmet", "@nestjs/throttler", "@nestjs/swagger", "@swc/core", "@swc/cli", "eslint-plugin-sonarjs", "husky", "lint-staged"]
  patterns: [swc-builder, eslint-flat-config, lint-staged-precommit]

key-files:
  created: ["nemea-back/package.json", "nemea-back/tsconfig.json", "nemea-back/tsconfig.build.json", "nemea-back/nest-cli.json", "nemea-back/eslint.config.mjs", "nemea-back/.prettierrc.json", "nemea-back/.lintstagedrc.json", "nemea-back/.husky/pre-commit", "nemea-back/src/main.ts", "nemea-back/src/app.module.ts", "nemea-back/src/app.controller.ts", "nemea-back/src/app.service.ts", "nemea-back/src/app.controller.spec.ts"]
  modified: ["nemea-back/.gitignore"]

key-decisions:
  - "Used NestJS 11 default scaffold with --strict flag, then enhanced tsconfig with full strict + path aliases"
  - "SWC builder with typeCheck: true for fast compilation + type safety"
  - "ESLint flat config (not legacy .eslintrc) matching frontend sonarjs/explicit-return-type conventions"
  - "Port 4000 set in main.ts matching CLAUDE.md backend convention"
  - "Replaced scaffolded .prettierrc with .prettierrc.json matching frontend (minus Tailwind/JSX plugins)"

patterns-established:
  - "ESLint flat config pattern: eslint.configs.recommended + tseslint.configs.recommended + sonarjs + custom rules"
  - "Pre-commit hook pattern: Husky -> lint-staged -> eslint --fix + prettier --check"
  - "Backend code style: single quotes, 2-space indent, trailing commas, explicit return types, no any"
  - "SWC builder pattern: nest-cli.json builder: swc with typeCheck: true"

requirements-completed: [INFR-01, INFR-05]

# Metrics
duration: 10min
completed: 2026-02-28
---

# Phase 1 Plan 01: NestJS Project Scaffold Summary

**NestJS 11 skeleton with SWC builder, TypeScript strict, ESLint flat config matching frontend conventions, and Husky pre-commit hooks**

## Performance

- **Duration:** 10 min
- **Started:** 2026-02-28T20:12:52Z
- **Completed:** 2026-02-28T20:22:59Z
- **Tasks:** 2
- **Files modified:** 17

## Accomplishments
- NestJS 11 project scaffolded in nemea-back/ with all core and dev dependencies installed
- SWC builder configured for fast hot reload with type checking enabled
- ESLint flat config enforces no-any, explicit-function-return-type, sonarjs/cognitive-complexity, and single quotes -- matching frontend conventions exactly
- Husky + lint-staged pre-commit pipeline blocks commits with lint/format errors (verified working during Task 2 commit)

## Task Commits

Each task was committed atomically:

1. **Task 1: Scaffold NestJS 11 project and install all dependencies** - `63ed2e1` (feat)
2. **Task 2: Configure ESLint flat config, Prettier, and Husky + lint-staged** - `5596319` (chore)

Note: Both commits are in the nemea-back/ sub-repo on the `development` branch.

## Files Created/Modified
- `nemea-back/package.json` - NestJS 11 project with all production + dev dependencies
- `nemea-back/tsconfig.json` - TypeScript strict mode with @src/* path aliases
- `nemea-back/tsconfig.build.json` - Build-specific TS config excluding spec files
- `nemea-back/nest-cli.json` - SWC builder with typeCheck enabled
- `nemea-back/eslint.config.mjs` - ESLint 9 flat config (sonarjs, no-any, explicit-return-type)
- `nemea-back/.prettierrc.json` - Prettier config matching frontend conventions
- `nemea-back/.lintstagedrc.json` - lint-staged config for .ts and .json files
- `nemea-back/.husky/pre-commit` - Pre-commit hook running lint-staged
- `nemea-back/.gitignore` - Updated with coverage/ and *.tsbuildinfo
- `nemea-back/src/main.ts` - NestJS bootstrap entry point (port 4000)
- `nemea-back/src/app.module.ts` - Root module
- `nemea-back/src/app.controller.ts` - Hello world controller with explicit return types
- `nemea-back/src/app.service.ts` - Hello world service with explicit return types
- `nemea-back/src/app.controller.spec.ts` - Controller unit test
- `nemea-back/test/app.e2e-spec.ts` - E2E test scaffold
- `nemea-back/test/jest-e2e.json` - Jest E2E configuration

## Decisions Made
- Used NestJS 11 scaffold default then enhanced rather than building from scratch -- faster and includes all NestJS conventions
- SWC builder with `typeCheck: true` -- gets fast SWC compilation while still running the type checker (important since SWC skips types by default)
- Port 4000 in main.ts -- matching CLAUDE.md convention (frontend on 3000, backend on 4000)
- Replaced the NestJS-generated ESLint config entirely -- the scaffold uses eslint-plugin-prettier/recommended with no-any: off, which conflicts with project conventions
- Kept ES2021 target in tsconfig per plan spec (scaffold used ES2023) -- wider runtime compatibility

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Scaffolded into temp directory due to existing nemea-back/.git**
- **Found during:** Task 1 (scaffold)
- **Issue:** `nest new` cannot scaffold into existing directory with .git. Plan anticipated this.
- **Fix:** Scaffolded to `nest-temp/` with `--skip-git`, then copied files into `nemea-back/` preserving .git
- **Files modified:** All nemea-back/ files
- **Verification:** .git directory preserved, commits on development branch
- **Committed in:** 63ed2e1

**2. [Rule 1 - Bug] Deleted scaffolded .prettierrc, created .prettierrc.json**
- **Found during:** Task 2 (Prettier config)
- **Issue:** NestJS scaffold creates `.prettierrc` (YAML format, minimal). Plan specifies `.prettierrc.json` with full config.
- **Fix:** Deleted `.prettierrc`, created `.prettierrc.json` with endOfLine, singleQuote, trailingComma, tabWidth, semi
- **Files modified:** .prettierrc (deleted), .prettierrc.json (created)
- **Verification:** `npm run prettier:check` passes
- **Committed in:** 5596319

---

**Total deviations:** 2 auto-fixed (1 blocking, 1 bug)
**Impact on plan:** Both were anticipated by the plan text. No scope creep.

## Issues Encountered
- node_modules copy from temp scaffold caused ENOTEMPTY errors during npm install -- resolved by removing node_modules and running clean `npm install`

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- NestJS 11 project ready for Plan 02 (Docker + environment config)
- All dependencies for TypeORM, config, validation, security already installed
- ESLint + Prettier + Husky toolchain operational and enforcing code style

## Self-Check: PASSED

All 13 created files verified on disk. Both commit hashes (63ed2e1, 5596319) verified in git log. SUMMARY.md exists.

---
*Phase: 01-backend-scaffold*
*Completed: 2026-02-28*
