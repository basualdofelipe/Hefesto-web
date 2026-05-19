---
status: awaiting_human_verify
trigger: "Backend crashes on startup with circular import between Role and User entities."
created: 2026-04-09T00:00:00Z
updated: 2026-04-09T00:01:00Z
---

## Current Focus
<!-- OVERWRITE on each update - reflects NOW -->

hypothesis: CONFIRMED AND FIXED
test: `npx nest build` — 0 issues, 133 files compiled
expecting: Backend starts without ReferenceError
next_action: Await human verification (npm run start:dev in real environment)

## Symptoms
<!-- Written during gathering, then IMMUTABLE -->

expected: Backend starts normally with `npm run start:dev`
actual: Crashes with "ReferenceError: Cannot access 'Role' before initialization" at role.entity.ts:7
errors: |
  ReferenceError: Cannot access 'Role' before initialization
      at Object.Role (nemea-back/src/roles/entities/role.entity.ts:7:14)
      at Object.<anonymous> (nemea-back/src/users/entities/user.entity.ts:21:23)
reproduction: Run `cd nemea-back && npm run start:dev` — crashes immediately after SWC compilation
started: Introduced by Phase 12.1 Plan 01 which added Role entity with @OneToMany(() => User) and User entity with @ManyToOne(() => Role, { eager: true }). TypeScript compiles fine (tsc --noEmit and nest build both pass with 0 errors). Only the Node.js runtime fails.

## Eliminated
<!-- APPEND only - prevents re-investigating -->

## Evidence
<!-- APPEND only - facts discovered -->

- timestamp: 2026-04-09T00:00:30Z
  checked: role.entity.ts line 3
  found: `import { User } from '../../users/entities/user.entity'` — hard runtime import
  implication: Node.js must evaluate user.entity.ts before Role class is initialized

- timestamp: 2026-04-09T00:00:30Z
  checked: user.entity.ts line 3
  found: `import { Role } from '../../roles/entities/role.entity'` — hard runtime import
  implication: Creates true circular ES module dependency: Role imports User imports Role

- timestamp: 2026-04-09T00:00:35Z
  checked: tsconfig.json
  found: emitDecoratorMetadata: true — rules out simple `import type` on both sides; runtime metadata requires class value
  implication: Fix must break the cycle on only one side (the @OneToMany side on Role), not both

- timestamp: 2026-04-09T00:00:40Z
  checked: scenario.entity.ts line 45
  found: `@OneToMany('ScenarioOverride', 'scenario', { cascade: true })` — existing codebase pattern for same problem
  implication: String entity name in @OneToMany + `import type` on the owning side is the established pattern in this project

- timestamp: 2026-04-09T00:01:00Z
  checked: npx nest build after fix
  found: 0 issues, 133 files compiled successfully
  implication: Fix is syntactically and type-safe correct

## Resolution
<!-- OVERWRITE as understanding evolves -->

root_cause: role.entity.ts and user.entity.ts form a hard circular ES module dependency. role.entity.ts imports User (runtime value), user.entity.ts imports Role (runtime value). When Node.js evaluates the module graph it must start one of them first; whichever executes second finds the other's export still `undefined` at the moment the class decorator runs, throwing "Cannot access 'Role' before initialization". TypeScript does not catch this because tsc only type-checks — it never simulates Node.js module evaluation order.
fix: In role.entity.ts — changed `import { User }` to `import type { User }` (erased at runtime, no module evaluation dependency) and changed `@OneToMany(() => User, ...)` to `@OneToMany('User', ...)` (TypeORM string-form entity reference, bypasses the need for the class value at decorator execution time). The User side keeps its direct `import { Role }` because the cycle is now one-directional at runtime. Matches the existing pattern in scenario.entity.ts line 45.
verification: `npx nest build` — 0 TypeScript issues, 133 files compiled with SWC
files_changed:
  - nemea-back/src/roles/entities/role.entity.ts
