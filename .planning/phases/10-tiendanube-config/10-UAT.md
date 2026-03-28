---
status: complete
phase: 10-tiendanube-config
source: [10-01-SUMMARY.md, 10-02-SUMMARY.md]
started: 2026-03-28T00:00:00.000Z
updated: 2026-03-28T00:00:00.000Z
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

[testing complete]

## Tests

### 1. Config page loads with seeded data
expected: Page loads at /configuracion/tiendanube showing all gateway sections with correct numeric rates from seed data
result: pass
previous: issue (blocker) — fixed in 10-03 gap closure
note: Plans show in alphabetical order (Escala, Esencial, Impulso, Inicial) instead of business order (Inicial→Esencial→Impulso→Escala). Future improvement: add sort_order column to tn_plans.

### 2. Gateway rate editing
expected: Click a rate value in any gateway section, change the number, click Guardar. The new value persists after page refresh.
result: pass

### 3. Installment rate editing
expected: Change a cuotas rate (e.g., 3 cuotas from 8.42 to 9.00), click Guardar. Value persists after refresh.
result: pass

### 4. Tax config editing
expected: Change IIBB rate, click Guardar. Value persists after refresh.
result: pass

### 5. Plan selector changes CPT display
expected: Change plan selector from Esencial to Impulso. CPT values in gateway sections update (Pago Nube stays 0%, others change from 2% to 1%).
result: pass

### 6. Verificar tasas link
expected: "Verificar tasas en Pago Nube" link at the bottom opens the official Pago Nube rates page in a new tab.
result: pass

### 7. Sidebar navigation
expected: Sidebar shows "Config Tiendanube" link under "Admin" section. Clicking navigates to /configuracion/tiendanube.
result: pass

### 8. Non-admin access blocked
expected: A USER-role account cannot access /configuracion/tiendanube (redirected to /).
result: pass

## Summary

total: 8
passed: 8
issues: 0
pending: 0
blocked: 0
skipped: 0

## Gaps

- truth: "Page loads at /configuracion/tiendanube showing all gateway sections with correct numeric rates from seed data"
  status: failed
  reason: "User reported: TypeError: Cannot read properties of null (reading 'toFixed') in GatewaySectionContent. ratePercent is null from raw SQL snake_case bug in parseGatewayRate/parseInstallmentRate/parseTaxConfig/parsePlan."
  severity: blocker
  test: 1
  artifacts:
    - nemea-back/src/tiendanube-config/tiendanube-config.service.ts
  missing:
    - "All parse helpers must read snake_case field names from raw SQL (rate_percent, iva_rate, iibb_rate, cpt_pago_nube, cpt_other_gateways) instead of camelCase"
