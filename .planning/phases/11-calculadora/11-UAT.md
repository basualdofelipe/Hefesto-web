---
status: partial
phase: 11-calculadora
source: [11-01-SUMMARY.md, 11-02-SUMMARY.md]
started: 2026-03-28T00:00:00.000Z
updated: 2026-03-28T00:00:00.000Z
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

number: 1
name: Calculadora page loads
expected: |
  Navigate to /calculadora. The page loads without errors and shows:
  - Two-column layout (inputs left, results right)
  - Mode toggle (Precio→Ganancia / Ganancia→Precio)
  - Product selector dropdown
  - Gateway selectors (Pasarela, Medio, Retiro, Cuotas)
  - Desglose panel (empty or with placeholder until calculation)
awaiting: user response

## Tests

### 1. Calculadora page loads
expected: Page loads at /calculadora without errors, showing two-column layout with inputs and desglose panel
result: issue
reported: "Maximum update depth exceeded. Infinite setState loop in CalculadoraClient/ProductSelector/GatewaySelectors. Page crashes on load. Stack trace points to SelectTrigger setRef in ProductSelector."
severity: blocker

### 2. Forward mode calculation
expected: Select a product (e.g., Billetera Hefesto 6 tarjetas), select Pago Nube + tarjeta + 7 días + 1 cuota. The desglose panel shows all 16 values (comisión, IVA, IIBB, ganancia real, margen) with ARS formatting.
result: blocked
blocked_by: prior-test
reason: "Page crashes on load (Test 1 blocker)."

### 3. Gateway cascading selectors
expected: Change pasarela to Mercado Pago. Medio de pago and Tiempo de retiro options update to match MP's options (not Pago Nube's). Select a combination and see recalculated results.
result: blocked
blocked_by: prior-test
reason: "Page crashes on load (Test 1 blocker)."

### 4. Inverse mode
expected: Switch to Ganancia→Precio mode. Enter a desired profit (e.g., $50,000). The desglose shows the required selling price. Switch back to forward mode with that price — the ganancia should match within $0.01.
result: blocked
blocked_by: prior-test
reason: "Page crashes on load (Test 1 blocker)."

### 5. Product cost auto-population
expected: When selecting a product that has a BOM with costs, the cost field auto-fills from the DB. Selecting a different product changes the cost.
result: blocked
blocked_by: prior-test
reason: "Page crashes on load (Test 1 blocker)."

### 6. Plan override toggle
expected: Toggle "Simular otro plan" — plan selector appears. Change plan. CPT values in desglose update accordingly.
result: blocked
blocked_by: prior-test
reason: "Page crashes on load (Test 1 blocker)."

### 7. Edge case: zero cost product
expected: Select a product with no BOM/cost ($0). In inverse mode, trying to calculate shows an error message "Definí el costo del producto primero" (not a crash).
result: blocked
blocked_by: prior-test
reason: "Page crashes on load (Test 1 blocker)."

### 8. Sidebar navigation
expected: Sidebar shows "Calculadora" under "Herramientas" section. Click navigates to /calculadora.
result: pass

## Summary

total: 8
passed: 1
issues: 1
pending: 0
blocked: 6
skipped: 0

## Gaps

- truth: "Page loads at /calculadora without errors, showing two-column layout with inputs and desglose panel"
  status: failed
  reason: "User reported: Maximum update depth exceeded. Infinite setState loop in CalculadoraClient/ProductSelector/GatewaySelectors. Stack trace points to SelectTrigger setRef causing recursive renders."
  severity: blocker
  test: 1
  artifacts:
    - nemea-front/src/app/(app)/calculadora/CalculadoraClient.tsx
    - nemea-front/src/components/calculadora/GatewaySelectors.tsx
    - nemea-front/src/components/calculadora/ProductSelector.tsx
  missing:
    - "State management in CalculadoraClient or GatewaySelectors causes infinite re-render loop. Likely a useMemo/useEffect dependency that triggers setState which triggers another useMemo recalculation."
