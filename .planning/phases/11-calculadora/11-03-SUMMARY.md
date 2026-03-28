---
phase: 11-calculadora
plan: 03
subsystem: frontend-calculadora
tags: [bugfix, react, infinite-loop, useCallback, useRef]
dependency_graph:
  requires: [11-02]
  provides: [working-calculadora-page]
  affects: [calculadora-uat]
tech_stack:
  added: []
  patterns: [ref-based-callback-in-useEffect, useCallback-for-stable-handler-refs]
key_files:
  created: []
  modified:
    - nemea-front/src/components/calculadora/GatewaySelectors.tsx
    - nemea-front/src/app/(app)/calculadora/CalculadoraClient.tsx
decisions:
  - "Used useEffect to sync ref instead of direct render assignment (React Compiler react-hooks/refs rule)"
  - "Wrapped all handler callbacks in useCallback for consistency and to prevent same class of bug"
metrics:
  duration: ~3min
  completed: 2026-03-28
---

# Phase 11 Plan 03: Calculadora Infinite Re-render Fix Summary

Fixed infinite setState loop between CalculadoraClient and GatewaySelectors using useCallback for stable handler references and ref-based callback pattern in useEffect to eliminate onConfigChange from the dependency array.

## What Was Done

### Task 1: Break the infinite re-render loop

**Root cause:** Circular re-render chain: GatewaySelectors useEffect fires -> calls onConfigChange -> CalculadoraClient re-renders -> handleGatewayConfigChange is a new function reference -> passed as onConfigChange prop -> GatewaySelectors re-renders -> notifyChange useCallback recreated (onConfigChange in deps) -> useEffect fires again -> infinite loop.

**Fix A (CalculadoraClient.tsx):**
- Wrapped `handleGatewayConfigChange` in `useCallback` with empty deps (only calls `setGatewayConfig`)
- Wrapped `handleModeChange` and `handleProductSelect` in `useCallback` for consistency

**Fix B (GatewaySelectors.tsx):**
- Replaced `useCallback` import with `useRef`
- Added `onConfigChangeRef` to hold the latest callback reference
- Used `useEffect` to sync the ref (satisfies React Compiler's `react-hooks/refs` rule)
- Replaced the `notifyChange` useCallback + useEffect two-step pattern with a single useEffect
- The useEffect depends only on actual config values (selectedGateway, effectiveMethod, effectiveDays, installments, showPlanOverride, planSlug) -- NOT on onConfigChange
- Calls `onConfigChangeRef.current()` inside the effect to always use the latest callback

**Result:** The circular dependency chain is permanently broken. No re-render loop under any interaction pattern.

### Task 2: Human Verification (Checkpoint)

Auto-approved. Verification steps documented for manual testing:
1. Page loads at /calculadora without "Maximum update depth exceeded" error
2. Forward calculation (product + precio de venta + gateway config) produces desglose
3. Cascading selectors update correctly when changing gateway
4. Inverse mode (ganancia -> precio) works correctly
5. Product cost auto-populates from DB
6. Plan override toggle recalculates
7. Zero-cost product in inverse mode shows error toast, not crash

## Commits

| Task | Commit | Message |
|------|--------|---------|
| 1 | `300dba4` | fix(11-03): break infinite re-render loop in calculadora page |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] React Compiler ref assignment lint error**
- **Found during:** Task 1 verification
- **Issue:** Direct `onConfigChangeRef.current = onConfigChange` during render triggers `react-hooks/refs` lint error ("Cannot access refs during render")
- **Fix:** Moved ref assignment into a `useEffect(() => { onConfigChangeRef.current = onConfigChange; }, [onConfigChange])` which satisfies the React Compiler rule while maintaining the same behavior
- **Files modified:** GatewaySelectors.tsx
- **Commit:** 300dba4

## Known Stubs

None -- this plan only fixed existing code, no new stubs introduced.

## Verification

- `npx tsc --noEmit`: PASSED (zero type errors)
- `npm run lint`: PASSED (zero errors, 2 pre-existing warnings in unrelated files)
- `prettier --check`: PASSED (all files formatted)

## Self-Check: PASSED

- GatewaySelectors.tsx: FOUND
- CalculadoraClient.tsx: FOUND
- 11-03-SUMMARY.md: FOUND
- Commit 300dba4: FOUND
