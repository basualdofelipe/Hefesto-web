---
phase: 04-supplies-and-price-history
plan: 03
subsystem: api, ui
tags: [nestjs, nextjs, react-hook-form, validation, conflict-exception, fragment-key, gap-closure]

# Dependency graph
requires:
  - phase: 04-supplies-and-price-history
    plan: 01
    provides: "SuppliesModule with toggleStatus, supplies.service.ts"
  - phase: 04-supplies-and-price-history
    plan: 02
    provides: "SupplyFormDialog, SupplyTypeGroup frontend components"
provides:
  - "toggleStatus with supplier.isActive validation guard on reactivation"
  - "Form reset on dialog reopen via useEffect watching [supply, open, reset]"
  - "Fragment key fix in SupplyTypeGroup map rendering"
affects: [05-products-and-bom]

# Tech tracking
tech-stack:
  added: []
  patterns: [conflict-exception-guard, form-reset-on-dialog-reopen]

key-files:
  created: []
  modified:
    - nemea-back/src/supplies/supplies.service.ts
    - nemea-front/src/components/supplies/SupplyFormDialog.tsx
    - nemea-front/src/components/supplies/SupplyTypeGroup.tsx

key-decisions:
  - "ConflictException guard checks both !supply.isActive AND !supply.supplier.isActive before blocking reactivation"
  - "useEffect deps include [supply, open, reset] to reset form both on prop change and dialog reopen"
  - "Fragment with explicit key replaces React shorthand <></> in map to fix console warning"

patterns-established:
  - "Supplier validation on supply reactivation: always check parent entity is_active before allowing child reactivation"
  - "Dialog form reset pattern: useEffect with open + entity + reset deps ensures fresh form data on every dialog open"

requirements-completed: [SPPL-01, SPPL-02, SPPL-03, SPPL-04, SPPL-05]

# Metrics
duration: 15min
completed: 2026-03-05
---

# Phase 4 Plan 3: Gap Closure Summary

**ConflictException guard on supply reactivation when supplier is inactive, form reset on dialog reopen via useEffect, and Fragment key fix eliminating console error**

## Performance

- **Duration:** 15 min
- **Started:** 2026-03-05T22:35:00Z
- **Completed:** 2026-03-05T22:50:00Z
- **Tasks:** 2 (1 auto + 1 checkpoint:human-verify)
- **Files modified:** 3

## Accomplishments
- Added ConflictException guard in toggleStatus preventing reactivation of supplies whose supplier is inactive
- Fixed stale edit modal data by adding useEffect with [supply, open, reset] deps that calls reset() on dialog reopen
- Fixed missing Fragment key in SupplyTypeGroup map rendering (was causing React console error)
- Phase 04 verification score moved from 4/7 to 7/7 -- all gaps closed

## Task Commits

Each task was committed atomically:

1. **Task 1a: Backend -- supplier validation on supply reactivation** - `3117d49` (fix) in nemea-back
2. **Task 1b: Frontend -- form reset + Fragment key fix** - `fc23d20` (fix) in nemea-front
3. **Task 2: Visual verification checkpoint** - approved by user

_Note: First frontend fix attempt (`939d8a3`) was reverted (`9111ac7`) and redone correctly in `fc23d20`._

## Files Created/Modified
- `nemea-back/src/supplies/supplies.service.ts` - Added ConflictException guard in toggleStatus checking supplier.isActive before allowing supply reactivation
- `nemea-front/src/components/supplies/SupplyFormDialog.tsx` - Added useEffect with [supply, open, reset] deps to reset form values on dialog reopen
- `nemea-front/src/components/supplies/SupplyTypeGroup.tsx` - Replaced keyless Fragment shorthand with `<Fragment key={...}>` in map rendering

## Decisions Made
- ConflictException guard condition uses `!supply.isActive && !supply.supplier.isActive` -- only blocks when reactivating (not deactivating) and only when supplier is also inactive
- useEffect deps include `open` (dialog state) in addition to `supply` and `reset` -- ensures form resets even when reopening with the same supply after cancel
- Fragment key fix preferred over restructuring the JSX -- minimal change, fixes the React console warning

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] First frontend fix attempt had incorrect behavior**
- **Found during:** Task 1 (frontend fix)
- **Issue:** Initial implementation (`939d8a3`) did not include `open` in useEffect deps, causing stale data after cancel-then-reopen
- **Fix:** Reverted via `9111ac7`, then committed correct version (`fc23d20`) with [supply, open, reset] deps
- **Files modified:** SupplyFormDialog.tsx
- **Verification:** User confirmed fresh data on reopen after cancel and after edit
- **Committed in:** `fc23d20`

**2. [Rule 1 - Bug] Missing Fragment key in map rendering**
- **Found during:** Task 1 (dev server error investigation -- Gap 2)
- **Issue:** SupplyTypeGroup used `<>...</>` shorthand inside a `.map()` call, causing React "missing key" console error
- **Fix:** Replaced with `<Fragment key={supply.id}>` import from react
- **Files modified:** SupplyTypeGroup.tsx
- **Verification:** Dev server console clean, no warnings
- **Committed in:** `fc23d20`

---

**Total deviations:** 2 auto-fixed (2 bugs)
**Impact on plan:** First deviation was a fix-iteration (reverted and redone). Second was the root cause of Gap 2 (dev server error). Both necessary for correctness.

## Issues Encountered
- Gap 2 (dev server error) turned out to be a missing Fragment key in SupplyTypeGroup.tsx, not a compilation error. The `tsc --noEmit` build passed because TypeScript does not check for React key warnings -- only the dev server runtime does.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Phase 04 is fully complete -- all 3 plans done, all 7 verification truths passing
- Supply CRUD, price history, and supplier cascade all verified end-to-end
- Ready for Phase 5: Products and BOM

## Self-Check: PASSED

- FOUND: commit 3117d49 in nemea-back (supplier validation fix)
- FOUND: commit fc23d20 in nemea-front (form reset + Fragment key fix)
- FOUND: commit 9111ac7 in nemea-front (revert of first attempt)
- VERIFIED: supplies.service.ts, SupplyFormDialog.tsx, SupplyTypeGroup.tsx modified
- All 3 verification gaps confirmed closed by user

---
*Phase: 04-supplies-and-price-history*
*Completed: 2026-03-05*
