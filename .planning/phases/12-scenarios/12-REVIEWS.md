---
phase: 12
reviewers: [gemini]
reviewed_at: "2026-03-29"
plans_reviewed: [12-01-PLAN.md, 12-02-PLAN.md]
---

# Cross-AI Plan Review — Phase 12

## Gemini Review

### Summary
The implementation plans for Phase 12 are exceptionally well-structured and demonstrate a deep understanding of the requirement to isolate simulation data from production pricing. The strategy of using a dedicated `ScenarioOverride` table ensures that "what-if" analysis never pollutes the `product_price_history`. The reuse of `CalculadoraModule` logic for forward calculations ensures consistency between the calculator and the scenarios. The frontend plan is particularly strong in its UX considerations, using color coding and status badges to provide immediate visual feedback on margin changes and product availability.

### Strengths
- **Data Isolation**: The architecture strictly separates scenario data from real product data, fulfilling the primary constraint of the phase.
- **Logical Reuse**: Leveraging the existing `calcForward()` logic ensures that the simulation uses the exact same math as the real system, maintaining "one source of truth" for margin calculations.
- **User Scoping**: The backend implementation correctly handles ownership and visibility (isPublic), ensuring that investors can collaborate without compromising private scenarios.
- **UX Feedback Loop**: The use of blue highlights for overridden values and green/red indicators for margin comparisons provides high "glanceability" for the user.
- **Bulk Operations**: The strategy for bulk adjustments (percentage by type + rounding) is pragmatic and aligns with how business users typically perform price increases.

### Concerns

#### Backend (Plan 12-01)
- **Transactional Integrity (MEDIUM)**: The `upsertOverrides` strategy uses a "delete all, then recreate" approach. Without being wrapped in a database transaction (`QueryRunner` or `@Transaction`), a failure after the delete but before the create would wipe a user's scenario data.
- **Performance Scaling (MEDIUM)**: The `calculate()` method loads all products, all costs, and all TN configurations synchronously. While fine for small catalogs, this O(N) operation may lead to timeouts or high memory usage if the product catalog grows significantly (e.g., >1000 items).
- **Redundant Calcs (LOW)**: The plan suggests calculating both "real" and "simulated" margins inside the loop. While safe, ensure the "real" calculation results aren't already available via a cached cost map to save CPU cycles.

#### Frontend (Plan 12-02)
- **Sequential API Latency (MEDIUM)**: `handleSaveAndCalculate` performs three sequential network requests (PUT overrides -> PUT config -> GET calculate). This creates a laggy "Save" experience and increases the risk of partial state updates if the tab is closed mid-process.
- **Render Performance (MEDIUM)**: Managing the override state as a single `Record<string, number>` in the parent component means that typing in one input might trigger a re-render of the entire 7-column table for all products.
- **Calculation Drift (LOW)**: The frontend performs bulk adjustments locally (`Math.round(basePrice * (1 + percentage / 100))`). Ensure this rounding logic perfectly matches the backend's expectations to avoid "off-by-one" discrepancies when the data is saved and re-fetched.

### Suggestions
1. **Atomic Upserts**: Wrap the `delete` and `save` operations for overrides in a transaction to prevent data loss on failure.
2. **Consolidated Save Endpoint**: Consider adding a "Save and Calculate" endpoint that accepts both config and overrides in a single request.
3. **Debounced Table Inputs**: Implement a debounced input component for the `ProductOverrideTable` to prevent UI stutter.
4. **Clear All Action**: Add a "Reset Scenario" or "Clear All Overrides" button (note: plan already has "Limpiar overrides" button).
5. **Pagination/Filtering**: If the catalog is large, add search/filter by product type in the ProductOverrideTable.

### Risk Assessment
**Level: LOW** — Plans are technically sound, risks are UX-related (latency) and edge-case data integrity (transactions), both easily addressable during implementation.

---

## Claude Pre-mortem (Senior Review)

Performed by `/senior pm` in the same session. Verified all plan assumptions against the actual codebase.

### Will Break
1. **findAll() excludes inactive products** (HIGH): `calculate()` calls `this.productsService.findAll()` without arguments. The actual signature is `findAll(includeInactive: boolean = false)` — only returns active products. CONTEXT.md says deactivated products should show as "(inactivo)" with preserved overrides. Fix: `findAll(true)`.
2. **Task 2a TypeScript verify will fail** (HIGH): ScenarioEditorClient imports ProductOverrideTable/BulkOverrideDialog/MarginSummary which don't exist until Task 2b. The `npx tsc --noEmit` verify step will fail in autonomous mode.

### Might Break
3. **upsertOverrides not transactional** (MEDIUM): Delete-then-insert without a transaction wrapper. Data loss if save fails after delete.
4. **GatewayPlanSelector cascading resets** (MEDIUM): Same prop-callback cascade pattern that caused Phase 11 infinite re-render bug. Plan doesn't adopt the defensive useCallback + useRef pattern from the fix.
5. **currentPrice string/number type confusion** (LOW): Backend returns decimal as string, frontend types it as number. Works via JS coercion but fragile for strict comparisons.
6. **taxConfig null produces confusing error** (LOW): calcForward's resolveRates throws generic NotFoundException if admin hasn't configured IVA/IIBB.

### Missing
7. No per-product error handling in calculate loop — one calcForward failure crashes entire calculation.
8. No scenario name editing in the editor page UI (API supports it, UI doesn't expose it).
9. findAll loads all overrides with relations for list page — potentially large payload just to show card count.

---

## Consensus Summary

### Agreed Concerns (raised by both reviewers)
1. **Transactional integrity for upsertOverrides** — Both Gemini and senior flagged the delete-then-insert pattern as a data loss risk. Wrap in a DB transaction. (MEDIUM)
2. **Performance with large catalogs** — Gemini flagged O(N) calculation scaling; senior flagged the findAll loading all overrides for the list page. (MEDIUM)
3. **Sequential API calls in handleSaveAndCalculate** — Gemini noted 3 sequential requests create latency risk. (MEDIUM)

### Senior-Only Concerns (codebase-verified)
4. **findAll() excludes inactive products** — Verified against `products.service.ts` actual signature. Guaranteed bug. (HIGH)
5. **Task 2a tsc verify will fail** — Structural task ordering issue. (HIGH)
6. **GatewayPlanSelector cascading state** — Same pattern as Phase 11 infinite re-render. (MEDIUM)

### Agreed Strengths
- Data isolation architecture (scenario_overrides never touches product_price_history)
- Reuse of calcForward() avoids formula duplication
- User scoping and visibility logic is well-designed
- UX color coding and visual feedback are strong

### Divergent Views
- **Risk level**: Gemini rated overall risk as LOW; senior rated FIX BEFORE EXECUTING due to the inactive products bug and task ordering issue (both verified against codebase). The difference is that Gemini reviewed the plan description while senior verified assumptions against actual code.
