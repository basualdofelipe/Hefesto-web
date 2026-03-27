---
created: 2026-03-26T00:00:00.000Z
title: Include CalculadoraService tests in Phase 11 plan
area: testing
files:
  - nemea-back/src/calculadora/calculadora.service.spec.ts
---

## Problem

CalculadoraService is the core of v1.1 — it calculates real money for investor decisions. No tests planned. If the formula is wrong, investors make decisions on false numbers.

## Solution

Phase 11 plan MUST include:
- Test calcForward with Billetera Hefesto at $87,000 (known values from prototype)
- Test calcInverse round-trip: forward(price) → profit, inverse(profit) → price, verify within $0.01
- Test edge cases: zero-cost product, very cheap product ($500), very expensive ($500,000)
- Test negative target profit in inverse mode (should return error, not garbage)
