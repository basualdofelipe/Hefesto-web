---
created: "2026-03-29T04:40:56.182Z"
title: Consistent circular ref pattern in scenario entities
area: database
files:
  - nemea-back/src/scenarios/entities/scenario-override.entity.ts:3,9
  - nemea-back/src/scenarios/entities/scenario.entity.ts:5,45
---

## Problem

scenario.entity.ts was fixed to use `import type` + string-based `@OneToMany('ScenarioOverride', 'scenario')` to break a circular dependency that crashed the backend on startup.

But scenario-override.entity.ts still uses a direct value import: `import { Scenario } from './scenario.entity'` and arrow-based `@ManyToOne(() => Scenario, ...)`. This works today because the cycle is already broken from the other side, but it's inconsistent — if someone adds another circular import between these two files, it will explode again.

## Solution

Change scenario-override.entity.ts to match the same pattern:
1. `import { Scenario }` → `import type { Scenario }`
2. `@ManyToOne(() => Scenario, (s) => s.overrides, ...)` → `@ManyToOne('Scenario', 'overrides', ...)`

One-liner fix, purely preventive.
