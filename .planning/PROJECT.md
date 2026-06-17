# Hefesto

## What This Is

App web para gestionar un emprendimiento de marroquinería (artículos de cuero). Permite ver productos con su costo real calculado, administrar insumos con historial de precios, registrar gastos, gestionar proveedores, simular pricing para Tiendanube, y ofrecer a inversores un dashboard con márgenes y escenarios. Reemplaza el Google Sheets actual con una app profesional. Para uso del dueño (admin) e inversores (read-only + simulador).

## Core Value

Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automáticamente cuando cambian los precios de los insumos.

## Requirements

### Validated

- ✓ Backend NestJS 11 + TypeORM + PostgreSQL + Docker + Swagger — v1.0 Phase 1
- ✓ Auth: Google OAuth + JWT + whitelist + roles (ADMIN/USER) — v1.0 Phase 2
- ✓ ABM catálogos (5 dimensiones producto + tipos insumo) — v1.0 Phase 3
- ✓ ABM proveedores con soft delete — v1.0 Phase 3
- ✓ ABM insumos con historial de precios append-only — v1.0 Phase 4
- ✓ ABM productos con SKU, BOM versionado, historial precios venta — v1.0 Phase 5
- ✓ Cálculo dinámico de costos (batched 2-query, DISTINCT ON) — v1.0 Phase 6
- ✓ Registro de gastos con categorías — v1.0 Phase 7
- ✓ TypeScript strict, zero `any`, explicit return types — v1.0
- ✓ Monorepo + sub-repos + 2-branch model — v1.0

### Active

- [ ] Middleware auth (proxy.ts no wired as middleware.ts — rutas frontend desprotegidas)
- [ ] 401 handling en apiClientFetch (redirect a login on JWT expiry)
- [ ] DRY cleanup: interfaces duplicadas (SupplyOption 8x, UNIT_LABELS 5x, formatDate 4x)
- [ ] Página admin de usuarios (backend CRUD existe, falta UI)
- [ ] Mejora página acceso-denegado (mensaje claro para inversores)
- [ ] Tipo insumo "producción externa" para productos de taller
- [ ] Agrupación jerárquica de productos: tipo → nombre → terminación
- [ ] BOM group editor scoped a nivel nombre (no tipo)
- [ ] Config Tiendanube: planes, tasas, cuotas, IIBB como tablas admin-editables
- [ ] Link "verificar tasas" a página oficial de Pago Nube
- [ ] Calculadora Tiendanube forward (precio → ganancia) con costos reales de DB
- [ ] Calculadora Tiendanube inverse (ganancia → precio)
- [ ] Dashboard inversores: tabla resumen catálogo con márgenes
- ✓ Simulador de escenarios guardables por usuario (override precios de venta) — v1.1 Phase 12
- ✓ Demo login env-gated (clone-and-run para reviewers) + readiness portfolio: suites 100% verdes, error/loading boundaries, constantes centralizadas — v1.1 Phase 12.5

### Out of Scope

- Integración API Tiendanube (lectura de ventas reales) — v2+, requiere OAuth con Tiendanube
- Clientes B2B y órdenes — v2, ventas fuera de Tiendanube
- Envíos (shippers, shipping_method) — v2, depende de clientes/órdenes
- Solicitudes de compra con aprobación por mail — v2+, sistema de workflows
- Migración de datos del Google Sheets — pendiente, primero la app
- Mobile app — web-first
- Notificaciones push/email — no hay necesidad actual
- Multi-idioma — app interna en español
- Corrección fiscal exacta — usuario consulta con contador, tasas configurables como workaround

## Current Milestone: v1.1 Tiendanube & Investor Dashboard

**Goal:** Hardening del v1, simulación de pricing para Tiendanube, y dashboard con escenarios para inversores.

**Target features:**
- Hardening: middleware, 401, DRY cleanup, users admin, producción externa
- Product UX: agrupación jerárquica, BOM grupal por nombre
- Config Tiendanube: tasas editables por admin
- Calculadora Tiendanube: forward/inverse con costos reales
- Simulador inversores: resumen catálogo + escenarios guardables

## Context


**Situación actual:** v1.0 completo — la app reemplaza Google Sheets para gestión de productos, insumos, costos y gastos. Desplegada en producción (Vercel + Railway). El negocio está pivotando de producción artesanal a compra de productos terminados de talleres externos.

**Modelo de productos:** tipo → nombre → terminación → color → talle (ej: Billetera Hefesto Lisa Marrón Grande). Cada producto tiene un SKU único y una composición de materiales (cueros en m², adicionales en unidades).

**Cálculo de costos:** `costo = SUM(último_precio(insumo) × cantidad)`. El último precio es el más reciente en `supplies_price_history`. La composición de materiales está en `supplies_per_product_history` donde `is_active = true`.

**Historial de precios:** Cuando se actualiza un precio de insumo se crea un nuevo registro. Los anteriores se conservan. El costo de producto se recalcula dinámicamente con el último precio.

**Modelo de datos:** Diseñado por el usuario en Lucidchart con mejoras de Claude aplicadas. 13 tablas para v1, ~13 más para v2.

**Referencia arquitectónica:** cuerpo-fit (frontend) y Freedom-Base (estructura de monorepo) del laburo del usuario.

**Archivos fuente:**
- `Diagrama ER db Hefesto.pdf` — modelo de datos original
- `Costos Hefesto.xlsx` — datos reales del negocio
- `calculadora/calculadora-tiendanube.jsx` — prototipo funcional (v2)
- `costos-hefesto-appscripts.txt` — lógica de Apps Scripts a migrar
- `.claude/docs/planning/business-rules.md` — reglas de negocio completas
- `.claude/docs/planning/scope-v1.md` — scope detallado de v1

## Constraints

- **Tech stack frontend**: Next.js 16 + TypeScript + Tailwind v4 + Shadcn/ui — ya scaffoldeado, no se cambia
- **Tech stack backend**: NestJS + TypeScript + TypeORM + PostgreSQL — decidido, patrón de Freedom-Base
- **Hosting**: Vercel (front) + Railway (back + DB) — decidido, ~$5/mes backend
- **Auth**: Google OAuth exclusivamente, sin passwords, whitelist de emails en DB — app interna con 2-3 usuarios
- **TypeORM**: Migrations siempre (dev y prod), nunca synchronize — práctica profesional
- **Docker**: Docker Compose para PostgreSQL local — igual que cuerpo-fit
- **Naming**: Inglés en DB y código (product_type, supplies, etc.)
- **TypeScript strict**: Sin `any`, explicit return types, single quotes

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Monorepo orquestador con sub-repos | Mismo patrón que Freedom-Base, cada proyecto tiene su git | ✓ Good |
| Google OAuth sin passwords | App interna, 2-3 usuarios conocidos, cero gestión de contraseñas | ✓ Good |
| Historial de precios (no JSONB) | Tabla normalizada con registros históricos, último precio = vigente | ✓ Good |
| Materiales normalizados (no JSONB) | supplies_per_product_history con unit_type permite queries y historial | ✓ Good |
| Soft delete (is_active) | Productos, insumos y proveedores con desactivación en vez de borrado físico | ✓ Good |
| Gastos con categorías | Registro básico con categorización para futuros reportes | ✓ Good |
| Timestamps en tablas principales | created_at/updated_at para auditoría en product, supplies, suppliers | ✓ Good |
| Cálculo de costo on-the-fly (no cache) | Batched 2-query con DISTINCT ON, performante para este volumen | ✓ Good |
| Pivot producción → talleres | BOM soporta "producción externa" como supply type sin cambios de schema | — Pending |
| Tasas Tiendanube configurables | Admin edita en DB, no hardcodeadas. Cálculo fiscal "estimado" — corrección exacta con contador | — Pending |
| Escenarios inversores en DB | USER crea scenarios con override de precios, no toca datos reales | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-26 after Phase 12.5 (demo login + portfolio code-readiness)*
