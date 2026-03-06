---
created: 2026-03-06T14:06:35.821Z
title: BOM dimension calculator - mm and circle support
area: ui
files:
  - nemea-front/src/components/products/BomEditorDialog.tsx
  - nemea-front/src/components/products/BomGroupEditorDialog.tsx
---

## Problem

When editing BOM for a product, the user must enter quantity in m² for leather supplies. This is unintuitive — the user thinks in millimeters (e.g., "300mm x 300mm for a wallet") not in square meters. Additionally, some products use circular cuts (e.g., posavasos with 9mm diameter circles), which makes the m² conversion even harder.

Currently the BOM editor just has a raw quantity input with the unit auto-filled from the supply type (m², unidad, metro, kg).

## Solution

For supplies with unit type `m2`, replace the plain quantity input with a dimension calculator:
- **Rectangle mode**: two inputs (width mm x height mm), auto-calculates m² (width * height / 1,000,000)
- **Circle mode**: one input (diameter mm), auto-calculates m² (pi * (d/2)² / 1,000,000)
- Toggle between rectangle/circle mode
- Show the calculated m² value but let the user think in mm
- Store the final m² value in the BOM (no schema change needed)
- Consider storing the raw dimensions as metadata for future reference (optional)
