[English](README.md) · **Español**

# Hefesto

Hefesto es una app web para gestionar y calcular costos, precios y ganancias de accesorios de cuero. A cada producto se le carga el detalle de costos (cuero, insumos, packaging, etc.) y una calculadora permite saber la ganancia según el precio de venta, contemplando costos + impuestos + comisiones + intereses. Reemplaza la planilla de Google Sheets con la que se llevaba todo esto, y suma config de Tiendanube, simulación de escenarios y un dashboard para inversores.

> **A destacar:** este es el repositorio *orquestador*. Enlaza los dos sub-proyectos —front y back—, que tienen su propio repositorio y se clonan dentro de este. El orquestador se encarga de correr **GSD (get-shit-done)**, un framework que estructura el desarrollo de aplicaciones fase por fase.

---

## Los dos sub-proyectos

| Proyecto | Repo | Qué hace | Stack | Link |
|----------|------|----------|-------|------|
| **Frontend** | `hefesto-front` | La web app: productos, costos, insumos, gastos, calculadora, escenarios, dashboard. Auth con Google + login demo. | Next.js 16 · TypeScript strict · Tailwind v4 · Shadcn/ui · Auth.js v5 | [→ repo](https://github.com/basualdofelipe/hefesto-front) |
| **Backend** | `hefesto-back` | API REST: lógica de negocio, motor de costos, motor de pricing, RBAC, persistencia. | NestJS 11 · TypeScript strict · TypeORM · PostgreSQL 16 · JWT | [→ repo](https://github.com/basualdofelipe/hefesto-back) |

```
┌──────────┐      ┌────────────────────┐      ┌────────────────────┐      ┌──────────────┐
│  Usuario │ ───► │  hefesto-front     │ ───► │  hefesto-back      │ ───► │  PostgreSQL  │
│ (browser)│      │  Next.js   · :3000 │      │  NestJS    · :4000 │      │   (Docker)   │
└──────────┘      └────────────────────┘      └────────────────────┘      └──────────────┘
                       Auth.js v5                  JWT + RBAC
                    (Google OAuth)             (11 permisos por rol)
```

Cada sub-repo tiene su propio README con setup, scripts, arquitectura interna y modelo de datos.

---

## Cómo trabaja GSD

Cada fase pasa por un pipeline con artefactos versionados que quedan en `.planning/`: se define *qué* hay que hacer, después *cómo*, se planifica el código, se revisa el plan, se ejecuta y se revisa lo construido.

### El ciclo

Arranca con `/new-project`, que genera un **MILESTONE** (un objetivo grande, ej. *"reemplazar el Google Sheets"*) y lo parte en **PHASES**. Cada fase recorre este ciclo:

```
                       ┌──────────────────────── MILESTONE ────────────────────────┐
     /new-project ───► │   objetivo grande  ──►  se divide en PHASES (1, 2, 3 …)    │
                       └─────────────────────────────┬──────────────────────────────┘
                                                      │
            ┌──────────────────  por cada PHASE  ─────┘
            ▼
   ╭───────────╮   ╭───────────╮   ╭───────────╮   ╭───────────────╮
   │ ① SPECS   │──►│ ② DISCUSS │──►│  ③ PLAN   │──►│ ④ PLAN REVIEW │
   │   ¿qué?   │   │  ¿cómo?   │   │  el plan  │   │  ¿está bien?  │
   ╰───────────╯   ╰───────────╯   ╰───────────╯   ╰───────┬───────╯
     criterios     decisiones      tarea por        ¿falta algo? ¿rompe?
     de éxito      de diseño       tarea                  │
                                                          │ ──► vuelve a ③ (re-planificar)
                                                          ▼ ok
   ╭───────────╮   ╭───────────────╮   ╭─────────────────╮
   │ ⑥ VERIFY  │◄──│ ⑤ CODE REVIEW │◄──│   EXECUTE       │
   │ ¿cumple   │   │  ¿hay bugs?   │   │  escribe código │
   │  el goal? │   │  ¿seguridad?  │   │  en front/back  │
   ╰───────────╯   ╰───────┬───────╯   ╰─────────────────╯
                     ¿bugs? │ ──► REVIEW-FIX (arregla y re-revisa)
```

El ciclo no es lineal: el *plan review* puede mandar a re-planificar antes de escribir código, y el *code review* puede disparar fixes (`REVIEW-FIX`) antes de cerrar la fase.

### Cada etapa ancla a un artefacto

Cada caja del gráfico es un archivo en `.planning/phases/<fase>/`:

| Etapa | Qué pasa | Artefacto |
|-------|----------|-----------|
| **① Specs** | Se define *qué* construir y los criterios de éxito. | `REQUIREMENTS.md` · `NN-SPEC.md` |
| **② Discuss** | Se decide *cómo* resolverlo. | `NN-CONTEXT.md` (+ `NN-RESEARCH.md`) |
| **③ Plan** | Un agente lee el repo real y propone un plan de código tarea por tarea. | `NN-PP-PLAN.md` |
| **④ Plan review** | Se audita el plan antes de codear: supuestos, pasos faltantes, regresiones. | `NN-REVIEWS.md` |
| **Execute** | Se ejecuta el plan y se escribe el código en el sub-repo, con commits atómicos. | `NN-PP-SUMMARY.md` |
| **⑤ Code review** | Se revisa lo escrito: bugs, seguridad, convenciones, tests. | `NN-REVIEW.md` |
| **⑥ Verify** | UAT contra los criterios de éxito de la fase. | `NN-VERIFICATION.md` / `NN-UAT.md` |

> `NN` = número de fase, `PP` = número de plan dentro de la fase. Una fase puede tener varios planes (la fase 5 tiene 4, la 12.4 tiene 8). No todas las fases tienen todos los artefactos, y la nomenclatura no es 100% uniforme entre fases.

### Lo que vive en `.planning/`

```
.planning/
├── PROJECT.md          # Contexto: qué es, para quién, restricciones, decisiones
├── ROADMAP.md          # Fases del milestone, estado y criterios de éxito
├── REQUIREMENTS.md     # Requirements con trazabilidad
├── STATE.md            # Estado actual del proyecto + métricas
├── research/           # Investigación previa al milestone
├── codebase/           # Mapeo del código existente
└── phases/
    └── 05-products-and-bom/
        ├── 05-CONTEXT.md            # ② discuss
        ├── 05-RESEARCH.md           # ② research
        ├── 05-01-PLAN.md  …         # ③ plan (uno por unidad de trabajo)
        ├── 05-01-SUMMARY.md  …      # execute
        └── 05-VERIFICATION.md       # ⑥ verify
```

### Ejemplo — Fase 5 (Products & BOM)

- **② Discuss** definió el formato de SKU `tipo.nombre.terminación.color.talle` → `1.2.1.3.0` (heredado del Google Sheets original), la creación batch por producto cartesiano de colores × talles, y el versionado del BOM con `is_active`.
- **③ Plan** se partió en 4 planes (entidad + migraciones → BOM + precios → tabla de productos → editores de UI), cada uno con los archivos exactos a tocar.
- **⑥ Verify** cerró con 5/5 criterios verificados, con evidencia a nivel de línea. Ej: al cambiar el BOM, el anterior se preserva con `is_active=false` — verificado en `products.service.ts:296-301`, dentro de una transacción.

Todo está en `.planning/phases/05-products-and-bom/`.

### En números

El proyecto pasó por **17 fases** y **54 planes ejecutados**, versionados en `.planning/`.

---

## Probarlo localmente

Clone-and-run con login demo: no hace falta configurar Google Cloud ni OAuth. Necesitás **Node 20+**, **npm**, **Docker** (para PostgreSQL) y dos terminales.

### 1. Cloná el orquestador y los dos repos adentro

```bash
git clone https://github.com/basualdofelipe/Hefesto-web.git
cd Hefesto-web
git clone https://github.com/basualdofelipe/hefesto-back.git
git clone https://github.com/basualdofelipe/hefesto-front.git
```

### 2. Backend + base de datos

```bash
cd hefesto-back
npm install
cp .env.example .env
docker compose up -d postgres     # PostgreSQL 16 en :5432
npm run start:dev                 # API en http://localhost:4000/api
```

Las migraciones corren solas al arrancar (`migrationsRun: true`), incluida la que siembra el usuario demo (`demo@hefesto.app`).

### 3. Frontend (otra terminal)

```bash
cd hefesto-front
npm install
cp .env.example .env.local
npm run dev                       # app en http://localhost:3000
```

El `.env.example` trae un `AUTH_SECRET` placeholder que arranca tal cual en local (si querés uno propio: `npx auth secret`).

### 4. Entrar

Abrí `http://localhost:3000` y usá el botón **"Entrar como demo"**: te loguea como `demo@hefesto.app` (rol admin, todos los permisos). El demo viene habilitado por defecto en los `.env.example` y se apaga en producción (`DEMO_LOGIN_ENABLED=false`).

> No hay instancia hosteada. La infra está lista para Vercel (front) + Railway (back + DB), pero el proyecto se corre local. Detalles de setup en el README de cada sub-repo.

---

## Roadmap

| Milestone | Alcance | Estado |
|-----------|---------|--------|
| **v1.0** | Reemplazo del Google Sheets: ABM de insumos/precios, productos/costos (BOM versionado), gastos y auth. | Completo |
| **v1.1** | Hardening, RBAC granular, UX de productos, config de Tiendanube, calculadora forward/inverse, escenarios y dashboard de inversores. | En progreso |
| **v2+** | Integración con la API de Tiendanube (ventas reales), clientes B2B, órdenes y compras a talleres. | Planeado |

Detalle por fase en [`.planning/ROADMAP.md`](.planning/ROADMAP.md).

---

## Estructura del repo

```
Hefesto-web/
├── .planning/        # GSD: specs, roadmap, fases, research, estado
├── CLAUDE.md         # Convenciones del proyecto
├── README.md         # versión en inglés (por defecto)
├── README.es.md      # este archivo
├── hefesto-front/    # se clona acá (repo aparte, gitignored)
└── hefesto-back/     # se clona acá (repo aparte, gitignored)
```
