# Phase 1: Backend Scaffold - Context

**Gathered:** 2026-02-28
**Status:** Ready for planning

<domain>
## Phase Boundary

NestJS 11 backend running locally (Docker Compose + PostgreSQL) and on Railway, with TypeORM migrations wired up, Swagger in dev, and all cross-cutting conventions baked in before any business entity is written. Frontend is already scaffolded — this phase delivers the backend foundation.

</domain>

<decisions>
## Implementation Decisions

### API response format
- Error responses follow consistent format: `{statusCode, error, message, timestamp}` via global HttpExceptionFilter
- Validation errors return message as array of individual field errors
- Global ValidationPipe with class-validator + class-transformer for automatic DTO validation
- Swagger UI only in development environment, accessible at `/api/docs`

### Claude's Discretion: Response envelope
- Success response format (envelope `{data, meta}` vs flat NestJS default) — Claude chooses what best fits the project

### Local dev setup
- Docker Compose obligatorio — PostgreSQL 16 only (no pgAdmin)
- `.env.example` completo con todos los valores de dev preconfigurados (copiar a .env y funciona)
- SWC compiler para hot reload rápido (`start:dev` usa SWC)
- Linting consistente con frontend: ESLint + Prettier, single quotes, 2 spaces, trailing commas, explicit return types, no `any`
- 4 comandos claros para arrancar: `docker compose up -d` → `npm install` → `npm run migration:run` → `npm run start:dev`
- Puerto 4000 (frontend en 3000)

### API prefix and routes
- Global prefix `/api` en todas las rutas
- Rutas en inglés: `/api/supplies`, `/api/products`, `/api/expenses`
- Sin versionado por ahora (no `/v1/`)
- Health check en `GET /api/health` devuelve `{status, app, version, uptime}`

### Rate limiting and security
- Throttler: 100 requests/minuto por IP
- CORS: solo `FRONTEND_URL` del .env (localhost:3000 en dev, dominio Vercel en prod)
- Helmet con config default activado
- Env validation con Joi: la app falla al arrancar si falta una variable requerida (error claro)
- Config Railway (Procfile/nixpacks) incluida desde el día 1
- Database SSL auto por ambiente: off en dev (Docker local), on en prod (Railway)

### Claude's Discretion: Logging
- Nivel de logging (NestJS default vs request logging completo) — Claude elige

### Testing strategy
- Unit tests (Jest + mocks) + E2E tests (supertest)
- DB separada para E2E: segundo servicio PostgreSQL en Docker Compose (puerto 5433, `nemea_test`)
- Sin threshold de cobertura obligatorio — reportar cobertura pero no bloquear

### Git hooks
- Husky + lint-staged: pre-commit corre lint + prettier check solo sobre archivos en staging
- Tests NO corren en pre-commit (se corren manualmente o en CI)

</decisions>

<specifics>
## Specific Ideas

- El scaffold debe ser production-ready desde el día 1: Railway deploy funcional con health check
- Consistencia total con el frontend en code style (mismas reglas de Prettier/ESLint)
- SWC compiler para que el hot reload sea rápido como Turbopack en el frontend

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `nemea-front/eslint.config.mjs`: Referencia para replicar reglas de linting en el backend (sonarjs, explicit-function-return-type, no-any)
- `nemea-front/.prettierrc.json`: Config de Prettier a copiar al backend (single quotes, trailing commas, 2 spaces)
- `nemea-front/.husky/pre-commit`: Referencia para hooks (`npm run lint && npm run prettier -- --check`)

### Established Patterns
- Path aliases: Frontend usa `@/*` → `src/*`. Backend debería usar el mismo patrón con `@src/*` o similar
- TypeScript strict mode: Habilitado en frontend, replicar en backend
- Import order: External → types → local → CSS (adaptado a NestJS)

### Integration Points
- `NEXT_PUBLIC_API_URL=http://localhost:4000` ya configurado en nemea-front — el backend debe responder en ese puerto
- Frontend espera consumir `/api/*` endpoints — el global prefix `/api` es consistente con esa expectativa
- `.env.example` del frontend ya referencia `NEXTAUTH_URL` y `NEXTAUTH_SECRET` — el backend necesitará `JWT_SECRET` y `FRONTEND_URL` para fase 2

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-backend-scaffold*
*Context gathered: 2026-02-28*
