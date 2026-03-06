---
created: 2026-03-06T18:51:54.031Z
title: Homepage product summary table with cost and price
area: ui
files:
  - nemea-front/src/app/(app)/page.tsx
---

## Problem

La página de inicio actualmente no muestra información útil. El usuario quiere ver un resumen rápido de productos seleccionados con datos clave (nombre, precio de venta, costo) sin tener que navegar a /productos y expandir cada uno.

## Solution

Agregar en la homepage una tabla resumen de productos seleccionados:
- Modal o selector para elegir qué productos mostrar en el dashboard
- Tabla con columnas: Nombre, Precio de venta, Costo (cuando exista el cálculo de costos)
- Posiblemente margen de ganancia como columna derivada
- UX a definir: puede ser un widget/card en el dashboard o una sección dedicada
- Depende de que Phase 6 (Cost Calculation) esté completa para mostrar costos reales
