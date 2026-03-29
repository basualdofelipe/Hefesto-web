---
phase: 12
slug: scenarios
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-03-28
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 29.x (backend) |
| **Config file** | `nemea-back/jest.config.ts` |
| **Quick run command** | `cd nemea-back && npx jest --testPathPattern=scenarios --no-coverage` |
| **Full suite command** | `cd nemea-back && npx jest --no-coverage` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd nemea-back && npx jest --testPathPattern=scenarios --no-coverage`
- **After every plan wave:** Run `cd nemea-back && npx jest --no-coverage`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 12-01-00 | 01 | 0 | SCEN-01, SCEN-03, SCEN-04 | scaffold | `test -f nemea-back/src/scenarios/scenarios.service.spec.ts` | W0 creates | ⬜ pending |
| 12-01-01 | 01 | 1 | SCEN-01 | unit | `npx jest --testPathPattern=scenarios.service` | ✅ W0 | ⬜ pending |
| 12-01-02 | 01 | 1 | SCEN-04 | unit | `npx jest --testPathPattern=scenarios.service` | ✅ W0 | ⬜ pending |
| 12-02-01 | 02 | 1 | SCEN-02 | unit | `npx jest --testPathPattern=scenarios.service` | ✅ W0 | ⬜ pending |
| 12-02-02 | 02 | 1 | SCEN-03 | unit | `npx jest --testPathPattern=scenarios.service` | ✅ W0 | ⬜ pending |
| 12-03-01 | 03 | 2 | SCEN-01 | manual | UI verification | N/A | ⬜ pending |
| 12-03-02 | 03 | 2 | SCEN-02 | manual | UI verification | N/A | ⬜ pending |
| 12-03-03 | 03 | 2 | SCEN-03 | manual | UI verification | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `nemea-back/src/scenarios/scenarios.service.spec.ts` — created by Plan 01, Task 0 with stubs for SCEN-01 (create+uniqueness), SCEN-03 (calculate delegation), SCEN-04 (user scoping)
- [ ] Test fixtures for scenario entities and override data (embedded in spec file)

*Existing test infrastructure (jest) covers framework needs.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Scenario editor UI with product table | SCEN-01 | Frontend component interaction | Create scenario, set override prices, verify table renders |
| Bulk adjustment applies correctly | SCEN-02 | Frontend + backend integration | Apply "+10% Cinturones", verify all cinturon prices updated |
| Margin summary table renders | SCEN-03 | Visual layout verification | Verify real vs simulated columns display correctly |
| Public scenario badge display | SCEN-04 | Visual verification | Toggle is_public, verify badge appears for other users |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (Plan 01 Task 0 creates spec file)
- [x] No watch-mode flags
- [x] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
