# Nemea

## What This Is

App web hub para gestionar un emprendimiento de marroquinería (artículos de cuero). Permite ver productos con su costo real calculado, administrar insumos con historial de precios, registrar gastos, y gestionar proveedores. Reemplaza el Google Sheets actual con una app profesional. Es para uso interno del dueño (admin) y en el futuro los inversores podrán ver reportes.

## Core Value

Saber el costo real y margen de ganancia de cada producto en todo momento, actualizado automáticamente cuando cambian los precios de los insumos.

## Requirements

### Validated

- ✓ Frontend scaffoldeado con Next.js 16 + Tailwind v4 + Shadcn/ui — existing (Fase 1)
- ✓ Estructura de monorepo con sub-repos independientes — existing
- ✓ Configuración de desarrollo (ESLint, Prettier, Jest, Husky) — existing
- ✓ Sistema de temas dark/light con next-themes — existing
- ✓ Tipografía custom (Engraving CC, Poppins, JetBrains Mono) — existing

### Active

- [ ] Backend NestJS scaffoldeado con TypeORM + PostgreSQL + Docker Compose
- [ ] Auth: Google OAuth via NextAuth (front) + JWT validation (back) + whitelist de emails
- [ ] ABM de insumos (cuero, herrajes, packaging) con tipo y proveedor
- [ ] Historial de precios por insumo (cada actualización = nuevo registro, el último es el vigente)
- [ ] ABM de proveedores con datos de contacto
- [ ] ABM de tipos de insumo
- [ ] ABM de productos con 5 dimensiones (tipo, nombre, terminación, color, talle)
- [ ] Soft delete en productos (is_active)
- [ ] Soft delete en insumos y proveedores (is_active)
- [ ] Composición de materiales por producto (insumos + cantidad + unit_type)
- [ ] Historial de composición de materiales (con is_active para cambios)
- [ ] Cálculo dinámico de costo por producto (último precio × cantidad de cada insumo)
- [ ] SKU único por producto (sku_code)
- [ ] ABM de catálogos (tipos de producto, nombres, terminaciones, colores, talles)
- [ ] Registro de gastos con categorías (materia prima, packaging, envío, etc.)
- [ ] Historial de precios de producto (precio de venta)
- [ ] Timestamps created_at/updated_at en tablas principales
- [ ] Roles: ADMIN (gestiona todo) y USER (solo lectura)

### Out of Scope

- Tiendanube (config, calculadora, integración API) — v2, el usuario lo descartó explícitamente de v1
- Dashboard para inversores — v2, no hay definición de métricas todavía
- Clientes B2B y órdenes — v2, ventas fuera de Tiendanube
- Envíos (shippers, shipping_method) — v2, depende de clientes/órdenes
- Solicitudes de compra con aprobación por mail — v2+, sistema de workflows
- Migración de datos del Google Sheets — última fase, primero la app
- Mobile app — web-first
- Notificaciones push/email — no hay necesidad actual
- Multi-idioma — app interna en español

## Context

**Empresa:** SEMPERGY ENTERPRISES SAS (marroquinería / artículos de cuero, marca Nemea)

**Situación actual:** Toda la gestión se hace en Google Sheets con Apps Scripts custom. Funciona pero es frágil, difícil de escalar, y no permite roles ni acceso controlado.

**Modelo de productos:** tipo → nombre → terminación → color → talle (ej: Billetera Hefesto Lisa Marrón Grande). Cada producto tiene un SKU único y una composición de materiales (cueros en m², adicionales en unidades).

**Cálculo de costos:** `costo = SUM(último_precio(insumo) × cantidad)`. El último precio es el más reciente en `supplies_price_history`. La composición de materiales está en `supplies_per_product_history` donde `is_active = true`.

**Historial de precios:** Cuando se actualiza un precio de insumo se crea un nuevo registro. Los anteriores se conservan. El costo de producto se recalcula dinámicamente con el último precio.

**Modelo de datos:** Diseñado por el usuario en Lucidchart con mejoras de Claude aplicadas. 13 tablas para v1, ~13 más para v2.

**Referencia arquitectónica:** cuerpo-fit (frontend) y Freedom-Base (estructura de monorepo) del laburo del usuario.

**Archivos fuente:**
- `Diagrama ER db Nemea.pdf` — modelo de datos original
- `Costos Nemea.xlsx` — datos reales del negocio
- `calculadora/calculadora-tiendanube.jsx` — prototipo funcional (v2)
- `costos-nemea-appscripts.txt` — lógica de Apps Scripts a migrar
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
| Google OAuth sin passwords | App interna, 2-3 usuarios conocidos, cero gestión de contraseñas | — Pending |
| Historial de precios (no JSONB) | Tabla normalizada con registros históricos, último precio = vigente | ✓ Good |
| Materiales normalizados (no JSONB) | supplies_per_product_history con unit_type permite queries y historial | ✓ Good |
| Soft delete (is_active) | Productos, insumos y proveedores con desactivación en vez de borrado físico | — Pending |
| Gastos con categorías | Registro básico con categorización para futuros reportes | — Pending |
| Timestamps en tablas principales | created_at/updated_at para auditoría en product, supplies, suppliers | — Pending |
| Tiendanube fuera de v1 | El usuario lo descartó explícitamente, se enfoca en costos y productos | ✓ Good |
| Cálculo de costo on-the-fly vs cache | Pendiente para cuando se implemente CostsModule | — Pending |

---
*Last updated: 2026-02-28 after initialization*
