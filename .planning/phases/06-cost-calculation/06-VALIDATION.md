---
phase: 6
slug: cost-calculation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-06
---

# Phase 6 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 29.x (backend), vitest (frontend) |
| **Config file** | `nemea-back/jest.config.ts`, `nemea-front/vitest.config.ts` |
| **Quick run command** | `npm test -- --testPathPattern=cost` |
| **Full suite command** | `npm test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test -- --testPathPattern=cost`
- **After every plan wave:** Run `npm test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 6-01-01 | 01 | 1 | COST-01 | unit | `npm test -- --testPathPattern=costs.service` | ❌ W0 | ⬜ pending |
| 6-01-02 | 01 | 1 | COST-02 | unit | `npm test -- --testPathPattern=costs.service` | ❌ W0 | ⬜ pending |
| 6-02-01 | 02 | 1 | COST-03 | integration | `npm test -- --testPathPattern=products.controller` | ❌ W0 | ⬜ pending |
| 6-02-02 | 02 | 1 | COST-04 | unit | `npm test -- --testPathPattern=products` | ❌ W0 | ⬜ pending |
| 6-03-01 | 03 | 2 | COST-01 | manual | visual inspection | N/A | ⬜ pending |
| 6-03-02 | 03 | 2 | COST-03 | manual | visual inspection | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `nemea-back/src/costs/costs.service.spec.ts` — stubs for COST-01, COST-02
- [ ] `nemea-back/src/products/products.controller.spec.ts` — stubs for COST-03, COST-04

*Existing infrastructure covers framework installation.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Cost column visible in product list | COST-01 | UI visual check | Navigate to /productos, verify cost column shows calculated values |
| Cost breakdown in product detail | COST-03 | UI visual check | Click a product, verify breakdown table with supply, qty, price, line cost |
| Cost updates after price change | COST-02 | E2E flow | Update a supply price, reload product list, verify cost changed |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
