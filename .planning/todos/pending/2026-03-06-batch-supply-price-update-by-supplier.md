---
created: 2026-03-06T18:45:00.622Z
title: Batch supply price update by supplier
area: ui
files:
  - nemea-front/src/app/(app)/insumos/page.tsx
  - nemea-back/src/supplies/supplies.controller.ts
  - nemea-back/src/supplies/supplies.service.ts
---

## Problem

When updating supply prices (e.g., after calling a tannery), the user gets all prices at once for multiple supplies from that supplier. Currently they have to update each supply's price one by one, which is tedious and error-prone. Real case: call Curtiembre Central, they give prices for 5 leather types — user wants to enter all 5 at once.

## Solution

Batch price update flow filtered by supplier:
- Select a supplier → see all their supplies with current prices
- Edit prices inline for multiple supplies at once
- Submit all price updates in a single batch operation
- Backend: batch endpoint that creates multiple price history records in a transaction
- UX to be designed later (could be a modal, a dedicated page, or inline in the supplies table)
