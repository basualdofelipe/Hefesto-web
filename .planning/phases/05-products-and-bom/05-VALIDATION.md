---
phase: 5
slug: products-and-bom
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-05
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (backend e2e) + Jest (frontend unit) |
| **Config file** | `nemea-back/test/jest-e2e.json` (backend) |
| **Quick run command** | `cd nemea-back && npx jest --config test/jest-e2e.json --testPathPattern products` |
| **Full suite command** | `cd nemea-back && npx jest --config test/jest-e2e.json` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Manual testing via Swagger (backend) and browser (frontend)
- **After every plan wave:** Run `cd nemea-back && npx jest --config test/jest-e2e.json`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 05-01-01 | 01 | 1 | PROD-01 | e2e | `npx jest --config test/jest-e2e.json --testPathPattern products` | No - W0 | pending |
| 05-01-02 | 01 | 1 | PROD-02 | e2e | Same as above | No - W0 | pending |
| 05-01-03 | 01 | 1 | PROD-03 | e2e | Same as above | No - W0 | pending |
| 05-01-04 | 01 | 1 | PROD-04 | e2e | Same as above | No - W0 | pending |
| 05-02-01 | 02 | 1 | PROD-05 | e2e | Same as above | No - W0 | pending |
| 05-02-02 | 02 | 1 | PROD-06 | e2e | Same as above | No - W0 | pending |
| 05-03-01 | 03 | 1 | PROD-07 | e2e | Same as above | No - W0 | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

- The project's established pattern does NOT include per-module e2e tests (only `app.e2e-spec.ts` exists)
- Validation relies on manual Swagger testing + `/gsd:verify-work` verification phase
- Frontend testing follows the minimal pattern (`__tests__/page.test.tsx` only)

*Existing infrastructure covers phase requirements via manual validation pattern.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Batch product creation UI flow | PROD-01 | Multi-step modal with previews | Create product via 3-step modal, verify all variants created |
| BOM editor Combobox UX | PROD-05 | UI interaction (grouped dropdown) | Open BOM editor, search/select supplies, verify grouping |
| BOM group editing | PROD-05/06 | Complex UI flow with majority detection | Select multiple products, edit BOM, verify versioning |
| Selling price inline add | PROD-07 | Inline UI component | Add price from expanded row, verify history dialog |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
