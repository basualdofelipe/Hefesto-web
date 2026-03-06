---
created: 2026-03-06T22:20:58.421Z
title: Add unit and integration testing
area: testing
files:
  - nemea-back/src/**/*.spec.ts
  - nemea-front/src/**/*.test.tsx
---

## Problem

El proyecto tiene tests unitarios mínimos solo en el backend (CostsService, algunos specs generados por NestJS) pero no hay una estrategia de testing definida. No hay tests de integración, no hay tests E2E, y el frontend no tiene ningún test. Para un proyecto portfolio/producción esto es una deuda técnica importante.

## Solution

- Backend: agregar tests unitarios para los services principales (Products, Supplies, Suppliers, Catalogs, Auth), tests de integración con DB de test (TypeORM + testcontainers o SQLite in-memory), y tests E2E para los flujos críticos (auth flow, CRUD completo, cost calculation)
- Frontend: agregar tests de componentes con React Testing Library + Vitest/Jest para los componentes clave (ProductTable, BOM editor, calculadora)
- Definir cobertura mínima target (ej: 70-80% en services del backend)
