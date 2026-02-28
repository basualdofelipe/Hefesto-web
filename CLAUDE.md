# Nemea-web

Monorepo orquestador para el proyecto Nemea — app de gestión y pricing para emprendimiento de marroquinería (cuero).

## Sub-proyectos

| Proyecto | Carpeta | Stack | Puerto | Descripción |
|----------|---------|-------|--------|-------------|
| Nemea Frontend | `nemea-front/` | Next.js + TypeScript + Tailwind v4 + Shadcn/ui | 3000 | Web app (calculadora, productos, dashboard) |
| Nemea Backend | `nemea-back/` | NestJS + TypeScript + PostgreSQL | 4000 | API REST + auth + lógica de negocio |

## Arquitectura

```
Usuario → nemea-front (Vercel) → nemea-back (Railway) → PostgreSQL (Railway)
```

- **Frontend**: Next.js 16+ con App Router, deployed en Vercel
- **Backend**: NestJS con TypeORM, deployed en Railway
- **Base de datos**: PostgreSQL en Railway
- **Auth**: NextAuth (frontend) + JWT (backend), múltiples usuarios con roles

## Branch Model

| Proyecto | Model | Branches |
|----------|-------|----------|
| Nemea-web (root) | Direct | `main` |
| nemea-front | 2-branch | `development` → `main` |
| nemea-back | 2-branch | `development` → `main` |

PRs en sub-proyectos siempre apuntan a `development`. Con tag `[PROD]` se promueve a `main`.

## Code Style

- **TypeScript strict** en todos los proyectos
- **No `any`** — siempre tipos explícitos
- **Explicit return types** en todas las funciones
- **Single quotes** para strings
- Correr `npm run lint` y `npm run prettier:fix` antes de commits
- Ver `.claude/docs/code-style.md` para más detalles

## Modelo de Datos (Referencia)

El negocio maneja:
- **Productos**: tipo → nombre → terminación → color (ej: Billetera Hefesto Lisa Marrón)
- **Insumos**: materiales con tipo (cuero, herraje, packaging) y proveedor
- **Precios**: historial de precios por insumo
- **Materiales por producto**: JSON con cueros (área en m²) y adicionales (cantidad)
- **Calculadora Tiendanube**: comisiones por plan/medio de pago/cuotas, IVA, retenciones

## Agentes Disponibles

| Agente | Uso |
|--------|-----|
| `plan-manager` | Crear, listar, archivar planes de implementación |
| `oop-coder` | Escribir código siguiendo OOP/SOLID |
| `oop-reviewer` | Revisar código contra SOLID y patrones del proyecto |
| `senior` | Consultor senior: reviews profundos, consulting técnico, validación |
| `code-simplifier` | Simplificar código (3 iteraciones automáticas) |
| `ui-ux` | Revisar diseño visual y experiencia de usuario |

## Comandos

| Comando | Acción |
|---------|--------|
| `/push` | Push + PR: `/push f d` (front→dev), `/push b p` (back→prod), `/push w` (web→main) |
| `/plan` | Gestionar planes (list, create, archive) |
| `/senior` | Consultor senior: `/senior r` (review), `/senior c` (consult) |
| `/run` | Levantar dev servers (front, back, o ambos) |
| `/stop` | Parar dev servers |
| `/chat-commit` | Commit solo los cambios de esta sesión |

## Políticas

Las políticas están en `.claude/docs/policies/`:
- **git-commit-policy.md** — Reglas de commits y branches
- **plan-execution-protocol.md** — Ciclo obligatorio por fase (code → commit → review → fix)
- **post-implementation-review.md** — Pipeline de review post-código

## Modelo de Permisos

- **Nunca** commitear sin pedido explícito del usuario
- **Nunca** hacer push sin confirmación
- **Nunca** usar `git add -A` sin revisar qué se stagea
- Incluir `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` en cada commit
