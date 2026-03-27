---
created: 2026-03-26T00:00:00.000Z
title: Define default plan/payment method for dashboard net margin column
area: ux
files: []
---

## Problem

Dashboard shows "net margin after Tiendanube deductions" but needs to assume a specific plan (Inicial/Esencial/Impulso/Escala) and payment method (tarjeta/transferencia) + deposit timing. Without a default, the margin number is meaningless.

## Solution

Add a selector at the top of the dashboard:
- Plan dropdown (default: whatever plan the business uses, e.g., Esencial)
- Payment method dropdown (default: Tarjeta, retiro 1 día)
- Cuotas dropdown (default: 1 cuota)

All products in the table recalculate margins with the selected config. Define this during Phase 13 context discussion.
