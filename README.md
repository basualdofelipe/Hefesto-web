# Nemea

App web de gestión y pricing para **Nemea** — emprendimiento de marroquinería en cuero. Reemplaza el Google Sheets que se usaba para insumos, productos, costos, gastos y la calculadora de pricing de Tiendanube. Incluye dashboard de inversores y escenarios de simulación de precios.

> Monorepo orquestador con dos sub-proyectos independientes (`nemea-front` + `nemea-back`).

---

## Stack

| Capa | Tecnología |
|------|------------|
| **Frontend** | Next.js 16 (App Router) · TypeScript · Tailwind v4 · Shadcn/ui · NextAuth |
| **Backend** | NestJS · TypeScript · TypeORM · JWT |
| **DB** | PostgreSQL (Railway) |
| **Hosting** | Vercel (front) · Railway (back + DB) |
| **Auth** | NextAuth (front) + JWT (back) — múltiples usuarios con roles dinámicos |

---

## Arquitectura

```
┌──────────┐      ┌──────────────────┐      ┌─────────────────┐      ┌────────────────┐
│  Usuario │ ───► │  nemea-front     │ ───► │  nemea-back     │ ───► │  PostgreSQL    │
│ (browser)│      │  Next.js · :3000 │      │  NestJS · :4000 │      │  Railway       │
└──────────┘      │  Vercel          │      │  Railway        │      └────────────────┘
                  └──────────────────┘      └─────────────────┘
                       NextAuth                  JWT + RBAC
```

- **Frontend** sirve UI + auth (Google OAuth via NextAuth) y llama al backend con el JWT.
- **Backend** expone REST, valida con guards (roles + permisos), y persiste vía TypeORM.
- **DB** centraliza todo: catálogos, productos, costos, historial de precios, configuración Tiendanube y escenarios.

---

## Sub-proyectos

| Proyecto | Carpeta | Stack | Puerto | Branch model |
|----------|---------|-------|--------|--------------|
| Frontend | `nemea-front/` | Next.js + TS + Tailwind v4 + Shadcn/ui | 3000 | `development` → `main` |
| Backend  | `nemea-back/`  | NestJS + TS + PostgreSQL + TypeORM     | 4000 | `development` → `main` |
| Root     | `./`           | Orquestador (este repo)                | —    | direct `main` |

Cada sub-repo tiene su propio README con detalles específicos de setup, scripts y deploy.

---

## Modelo de datos

> Generado a partir de las entities TypeORM en `nemea-back/src/**/*.entity.ts`.
> Todas las entidades extienden `BaseEntity` → `id: uuid PK`, `created_at: timestamptz`, `updated_at: timestamptz` (omitidos en los diagramas para que se vean limpios).
>
> Se divide en 5 dominios. Cada diagrama es independiente y autocontenido.

### 1. Auth & RBAC

Usuarios con login por Google y roles dinámicos con permisos granulares.

```mermaid
erDiagram
    USERS {
        uuid id PK
        varchar email UK
        varchar name
        varchar picture_url
        varchar google_id
        uuid role_id FK
        bool is_active
    }
    ROLES {
        uuid id PK
        varchar name UK
        varchar description
        bool is_system
        bool can_view_products
        bool can_edit_products
        bool can_view_supplies
        bool can_edit_supplies
        bool can_view_expenses
        bool can_edit_expenses
        bool can_use_calculator
        bool can_manage_scenarios
        bool can_view_dashboard
        bool can_manage_config
        bool can_manage_users
    }
    USERS }o--|| ROLES : ""
```

### 2. Suppliers & Supplies

Proveedores, insumos (cuero, herrajes, packaging) con tipo y unidad, e historial inmutable de precios.

```mermaid
erDiagram
    SUPPLIERS {
        uuid id PK
        varchar name
        varchar address
        varchar email
        varchar phone
        varchar whatsapp
        text description
        bool is_active
    }
    SUPPLY_TYPES {
        uuid id PK
        varchar name UK
    }
    SUPPLIES {
        uuid id PK
        varchar name
        uuid type_id FK
        uuid supplier_id FK
        enum unit_type "m2|unidad|metro|kg"
        text notes
        bool is_active
    }
    SUPPLY_PRICE_HISTORY {
        uuid id PK
        uuid supply_id FK
        decimal price
    }
    SUPPLIES }o--|| SUPPLY_TYPES : ""
    SUPPLIES }o--|| SUPPLIERS : ""
    SUPPLY_PRICE_HISTORY }o--|| SUPPLIES : ""
```

### 3. Products Catalog

Productos con SKU compuesto por 5 ejes (type · name · finish · color · size), historial de precios e historial de BOM (insumos por producto).

```mermaid
erDiagram
    PRODUCT_TYPES {
        uuid id PK
        varchar name UK
        smallint sku_code UK
    }
    PRODUCT_NAMES {
        uuid id PK
        varchar name UK
        smallint sku_code UK
    }
    PRODUCT_FINISHES {
        uuid id PK
        varchar name UK
        smallint sku_code UK
    }
    PRODUCT_COLORS {
        uuid id PK
        varchar name UK
        smallint sku_code UK
    }
    PRODUCT_SIZES {
        uuid id PK
        varchar name UK
        smallint sku_code UK
        smallint sort_order
    }
    PRODUCTS {
        uuid id PK
        varchar sku_code
        uuid product_type_id FK
        uuid product_name_id FK
        uuid product_finish_id FK
        uuid product_color_id FK
        uuid product_size_id FK
        bool is_active
    }
    PRODUCT_PRICE_HISTORY {
        uuid id PK
        uuid product_id FK
        decimal price
        varchar currency "default ARS"
    }
    SUPPLIES_PER_PRODUCT_HISTORY {
        uuid id PK
        uuid product_id FK
        uuid supply_id FK
        decimal quantity
        bool is_active
    }
    SUPPLIES {
        uuid id PK
        varchar name
    }
    PRODUCTS }o--|| PRODUCT_TYPES : ""
    PRODUCTS }o--|| PRODUCT_NAMES : ""
    PRODUCTS }o--|| PRODUCT_FINISHES : ""
    PRODUCTS }o--|| PRODUCT_COLORS : ""
    PRODUCTS }o--|| PRODUCT_SIZES : ""
    PRODUCT_PRICE_HISTORY }o--|| PRODUCTS : ""
    SUPPLIES_PER_PRODUCT_HISTORY }o--|| PRODUCTS : ""
    SUPPLIES_PER_PRODUCT_HISTORY }o--|| SUPPLIES : ""
```

> `SUPPLIES` aparece como stub para mostrar la relación de la BOM. Su definición completa está en el diagrama (2).

### 4. Expenses

Gastos con categoría. Simple, sin historial — se versiona con `updated_at`.

```mermaid
erDiagram
    EXPENSE_CATEGORIES {
        uuid id PK
        varchar name UK
    }
    EXPENSES {
        uuid id PK
        decimal amount
        varchar concept
        date date
        uuid category_id FK
    }
    EXPENSES }o--|| EXPENSE_CATEGORIES : ""
```

### 5. Tiendanube Config & Scenarios

Configuración de planes, gateways, cuotas e impuestos para la calculadora de pricing. Los escenarios permiten guardar combinaciones y sobreescribir precios de productos puntuales sin tocar el catálogo real.

```mermaid
erDiagram
    TN_PLANS {
        uuid id PK
        varchar slug UK
        varchar label
        decimal cpt_pago_nube
        decimal cpt_other_gateways
        bool only_pago_nube
        bool is_active
    }
    TN_PAYMENT_GATEWAYS {
        uuid id PK
        varchar slug UK
        varchar label
        bool is_active
    }
    TN_GATEWAY_RATES {
        uuid id PK
        uuid gateway_id FK
        varchar payment_method
        int withdrawal_days
        decimal rate_percent
        bool is_active
    }
    TN_INSTALLMENT_RATES {
        uuid id PK
        int installments
        decimal rate_percent
        bool is_active
    }
    TN_TAX_CONFIG {
        uuid id PK
        decimal iva_rate
        decimal iibb_rate
        bool is_active
    }
    SCENARIOS {
        uuid id PK
        varchar name
        uuid user_id FK
        bool is_public
        varchar gateway_slug
        varchar payment_method
        int withdrawal_days
        int installments
        uuid plan_id FK "nullable"
    }
    SCENARIO_OVERRIDES {
        uuid id PK
        uuid scenario_id FK
        uuid product_id FK
        decimal override_price
    }
    USERS {
        uuid id PK
        varchar email
    }
    PRODUCTS {
        uuid id PK
        varchar sku_code
    }
    TN_GATEWAY_RATES }o--|| TN_PAYMENT_GATEWAYS : ""
    SCENARIOS }o--|| USERS : ""
    SCENARIOS }o--o| TN_PLANS : ""
    SCENARIO_OVERRIDES }o--|| SCENARIOS : ""
    SCENARIO_OVERRIDES }o--|| PRODUCTS : ""
```

> `USERS` y `PRODUCTS` aparecen como stubs para mostrar las relaciones cruzadas.

### Notas de modelado

- **UUIDs en todas las entidades** (`@PrimaryGeneratedColumn('uuid')`).
- **Roles dinámicos con permisos granulares**: cada permiso es un boolean en `roles`. Los roles `is_system = true` no se pueden borrar.
- **SKU compuesto** en `products`: se arma con los `sku_code` de los 5 catálogos (type · name · finish · color · size).
- **Historial inmutable** (`*_price_history`, `supplies_per_product_history`): nunca se updatea, se insertan nuevas filas. El precio/BOM "actual" es la fila más reciente. `ON DELETE CASCADE` desde el producto/insumo padre.
- **Scenarios** permiten simular pricing alternativo (gateway, plan, cuotas) y sobreescribir precios de productos específicos sin tocar el catálogo real.
- **Tiendanube config** está desnormalizada en varias tablas para soportar combinaciones de plan × gateway × medio de pago × cuotas × días de retiro.

---

## Quickstart

```bash
# Frontend
cd nemea-front
npm install
npm run dev          # http://localhost:3000

# Backend (en otra terminal)
cd nemea-back
npm install
npm run start:dev    # http://localhost:4000
```

> Variables de entorno en cada sub-repo (`.env.example`). Para DB local usar PostgreSQL o apuntar al Railway dev.

---

## Comandos del orquestador

| Comando | Acción |
|---------|--------|
| `/run`  | Levantar dev servers (front, back, o ambos) |
| `/stop` | Parar dev servers |
| `/commit` | Commit local: `/commit f`, `/commit b "msg"`, `/commit w` |
| `/push` | Push + PR: `/push f d` (front→dev), `/push b p` (back→prod) |
| `/plan` | Gestionar planes de implementación |
| `/senior` | Senior code reviewer (review, chat, audit, premortem) |

---

## Estructura del repo

```
Nemea-web/
├── nemea-front/          # Next.js app
├── nemea-back/           # NestJS API
├── calculadora/          # Prototipo original de la calculadora (referencia)
├── .planning/            # GSD: PROJECT, ROADMAP, STATE, fases, research
├── .claude/              # Agentes, commands, docs internos
└── CLAUDE.md             # Orquestador raíz (convenciones del monorepo)
```

---

## Roadmap

- **v1.0** — ✅ Reemplazo de Google Sheets: ABM insumos/precios, productos/costos, gastos, auth.
- **v1.1** — 🚧 Hardening, UX de productos, config Tiendanube, calculadora integrada, dashboard de inversores.
- **v2+** — Integración API Tiendanube (ventas reales), clientes B2B, órdenes, solicitudes de compra a talleres.

Detalle completo en [`.planning/ROADMAP.md`](.planning/ROADMAP.md).
