---
phase: 7
slug: expenses-and-config
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-06
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (backend e2e via NestJS) |
| **Config file** | nemea-back/test/jest-e2e.json |
| **Quick run command** | `cd nemea-back && npm run lint && npm run build` |
| **Full suite command** | `cd nemea-back && npm run test:e2e` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd nemea-back && npm run lint && npm run build`
- **After every plan wave:** Run `cd nemea-back && npm run test:e2e`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 1 | EXPN-03 | e2e | `cd nemea-back && npx jest --config test/jest-e2e.json test/catalogs.e2e-spec.ts` | Existing (extend) | pending |
| 07-02-01 | 02 | 1 | EXPN-01 | e2e | `cd nemea-back && npx jest --config test/jest-e2e.json test/expenses.e2e-spec.ts` | Wave 0 | pending |
| 07-02-02 | 02 | 1 | EXPN-02 | e2e | `cd nemea-back && npx jest --config test/jest-e2e.json test/expenses.e2e-spec.ts` | Wave 0 | pending |

*Status: pending · green · red · flaky*

---

## Wave 0 Requirements

- [ ] `nemea-back/test/expenses.e2e-spec.ts` — stubs for EXPN-01, EXPN-02
- [ ] Extend existing `nemea-back/test/catalogs.e2e-spec.ts` with expense-categories dimension — covers EXPN-03

*Existing infrastructure covers test framework setup.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Month-grouped collapsible UI | EXPN-02 | Visual layout verification | Open /finanzas/gastos, verify expenses grouped by month with collapsible sections, subtotals per month |
| Category colored badges | EXPN-02 | Visual styling | Verify each category shows a distinct colored badge |
| Summary bar dynamic updates | EXPN-02 | Interactive UI behavior | Apply filters, verify summary bar updates with filtered totals |
| Sidebar navigation order | EXPN-02 | Layout verification | Verify sidebar order: Inicio > Finanzas (Gastos) > Productos > Datos base |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
