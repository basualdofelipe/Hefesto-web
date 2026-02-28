# Stack Research

**Domain:** Inventory & cost management backend — leather goods SMB
**Researched:** 2026-02-28
**Confidence:** MEDIUM-HIGH (core stack HIGH, version specifics MEDIUM)

---

## Summary

The stack is already decided (NestJS + TypeORM + PostgreSQL). This research validates that
decision, fills in the specific library versions, documents the exact auth integration pattern,
and identifies the gaps in the original stack definition that need to be addressed in scaffolding.

**Bottom line:** The chosen stack is correct and production-grade. No changes recommended.
The biggest gap is the auth integration between NextAuth v4 (frontend JWT) and NestJS (backend
JWT validation). This pattern is well-established but requires careful implementation.

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| NestJS | ^11.x (11.1.14 latest) | Backend framework | Current stable. NestJS 11 released Jan 2025 with improved module startup perf and structured logging. Maintained until 2030+. |
| TypeORM | ^0.3.28 | ORM for PostgreSQL | Current stable. The 0.3.x API (DataSource) is the active API. No planned major breaking changes. |
| PostgreSQL | 16-alpine (Docker) | Primary database | 16 is the current stable with query plan improvements. Alpine image keeps Docker layer small. |
| Node.js | 20 LTS | Runtime | Already used by nemea-front. 20 LTS is supported until April 2026; Node 22 LTS available as upgrade path. |
| TypeScript | ~5.x (matches NestJS 11) | Type safety | Already decided. Strict mode required. |

### Authentication Stack

| Library | Version | Purpose | Why This |
|---------|---------|---------|----------|
| @nestjs/passport | ^11.0.5 | Passport integration for NestJS | Official NestJS adapter. Version 11 matches NestJS 11 core. |
| passport | ^0.7.x | Authentication middleware | Required peer dependency of @nestjs/passport. |
| passport-jwt | ^4.x | JWT bearer token strategy | Validates JWT from Authorization header. v5 is pending (has breaking changes) — use v4 for now. |
| @nestjs/jwt | ^10.x | JWT signing and verification | Official NestJS JWT module. Wraps jsonwebtoken. |
| passport-google-oauth20 | ^2.x | Google OAuth2 strategy for Passport | Used on backend only if doing server-side OAuth flow. For the frontend-driven model (NextAuth handles OAuth, backend only validates JWT), this is NOT needed. |

**Auth architecture for this project:**
NextAuth (frontend) handles the full Google OAuth flow and issues an encrypted JWT stored
in a cookie. The NestJS backend does NOT need to handle Google OAuth redirects. The backend
only needs to:
1. Expose `POST /auth/validate` — receives the NextAuth JWT token, verifies signature using
   the shared `NEXTAUTH_SECRET`, looks up email in the users whitelist table, returns user data.
2. Expose `GET /auth/me` — protected route returning current user from JWT in Authorization header.

This means `passport-google-oauth20` is NOT needed on the backend. Use only `passport-jwt`
with a custom `JwtStrategy` that validates the NextAuth-signed token.

### Data Access Layer

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| pg | ^8.x | PostgreSQL driver (node-postgres) | Required peer dependency of TypeORM for Postgres. TypeORM does not bundle it. |
| @nestjs/typeorm | ^11.x | TypeORM module for NestJS | Official integration. Version matches NestJS 11. |

**On TypeORM DECIMAL columns:** TypeORM returns DECIMAL values as strings from PostgreSQL.
For cost calculation aggregates (SUM of price * quantity), use QueryBuilder and parse results
with `parseFloat()` at the service layer, or use a column transformer. Do NOT rely on automatic
numeric coercion. This is a known TypeORM quirk, not a bug.

### Validation & Serialization

| Library | Version | Purpose | Why This |
|---------|---------|---------|----------|
| class-validator | ^0.14.x | DTO input validation | Standard NestJS validation. Use standalone package (not @nestjs/class-validator which is 4yr old fork). |
| class-transformer | ^0.5.x | DTO serialization / plainToClass | Works with ValidationPipe and @Exclude decorators for response shaping. Use standalone package. |

### Configuration

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| @nestjs/config | ^3.x | Environment variable management | Official module. Use `isGlobal: true` + `validationSchema` (Joi) for env var validation at startup. Fails fast if required vars are missing. |
| joi | ^17.x | Schema validation for env vars | Use with @nestjs/config's `validationSchema`. Validates DATABASE_URL, JWT_SECRET, PORT, etc. at boot time. |

### API Documentation

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| @nestjs/swagger | ^11.2.6 | OpenAPI / Swagger docs | Latest version (11.2.6 published Feb 2026). Auto-generates API docs from decorators. Use in development; can disable in production via env flag. |
| swagger-ui-express | (peer dep) | Swagger UI server | Installed automatically with @nestjs/swagger for Express adapter. |

### Security

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| helmet | ^8.x | HTTP security headers | Use `app.use(helmet())` in main.ts before all routes. Sets X-Frame-Options, CSP, etc. |
| @nestjs/throttler | ^6.x | Rate limiting | Official NestJS rate limiting module. Apply at global level with `ThrottlerGuard`. |

### Development Tools

| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| @nestjs/cli | ^11.x | NestJS CLI (generate, build, start) | Required for `nest generate` scaffolding commands. Install globally. |
| ts-node | ^10.x | TypeScript execution for CLI | Required by TypeORM CLI to read `dataSource.ts` directly. |
| tsconfig-paths | ^4.x | Path alias resolution at runtime | Required so TypeORM CLI can resolve `@/*` aliases in `dataSource.ts`. |

### Testing

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| @nestjs/testing | ^11.x | NestJS test utilities | Creates test modules. Standard for unit tests. |
| jest | ^29.x | Test runner | NestJS CLI scaffolds this. Use ts-jest preset. |
| supertest | ^7.x | HTTP integration testing | For e2e tests against the live NestJS app instance. |
| @types/supertest | ^6.x | TypeScript types for supertest | Dev dependency. |

---

## Installation

```bash
# Inside nemea-back/

# Core framework
npm install @nestjs/common @nestjs/core @nestjs/platform-express reflect-metadata rxjs

# Database
npm install @nestjs/typeorm typeorm pg

# Auth
npm install @nestjs/passport @nestjs/jwt passport passport-jwt

# Configuration
npm install @nestjs/config joi

# Validation
npm install class-validator class-transformer

# Security
npm install helmet @nestjs/throttler

# API Docs
npm install @nestjs/swagger

# Dev dependencies
npm install -D @nestjs/cli @nestjs/testing ts-node tsconfig-paths jest ts-jest supertest @types/jest @types/supertest @types/passport-jwt
```

---

## Docker Compose (Local PostgreSQL)

```yaml
# docker-compose.yml in nemea-back/
services:
  postgres:
    image: postgres:16-alpine
    container_name: nemea-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: nemea
      POSTGRES_PASSWORD: nemea_dev
      POSTGRES_DB: nemea_db
    volumes:
      - nemea_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nemea -d nemea_db"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  nemea_pgdata:
```

Run NestJS outside Docker in development (just `npm run start:dev`). Only PostgreSQL runs in
Docker locally. This matches the cuerpo-fit reference pattern.

---

## TypeORM Migration Setup

This is the most critical configuration decision. Migrations always, never synchronize.

```typescript
// src/database/dataSource.ts (used by TypeORM CLI — runs WITHOUT NestJS)
import { DataSource } from 'typeorm';
import { config } from 'dotenv';
import { join } from 'path';

config(); // Load .env

export const AppDataSource = new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production'
    ? { rejectUnauthorized: false }
    : false,
  entities: [join(__dirname, '..', '**', '*.entity.{ts,js}')],
  migrations: [join(__dirname, '..', 'migrations', '*.{ts,js}')],
  synchronize: false,  // NEVER true. Not in dev. Not in prod.
  logging: process.env.NODE_ENV !== 'production',
});
```

```json
// package.json scripts
{
  "typeorm": "ts-node -r tsconfig-paths/register ./node_modules/typeorm/cli",
  "migration:generate": "npm run typeorm -- migration:generate src/migrations/$npm_config_name -d src/database/dataSource",
  "migration:run": "npm run typeorm -- migration:run -d src/database/dataSource",
  "migration:revert": "npm run typeorm -- migration:revert -d src/database/dataSource"
}
```

Usage:
```bash
npm run migration:generate --name=CreateSuppliesTable
npm run migration:run
```

---

## Auth Integration: NextAuth → NestJS JWT

This is the key architectural point specific to this project.

**Flow:**
1. User clicks "Sign in with Google" on Next.js frontend.
2. NextAuth handles Google OAuth and creates a session JWT signed with `NEXTAUTH_SECRET`.
3. NextAuth JWT is stored in an HTTP-only cookie on the frontend domain.
4. Frontend sends requests to NestJS with the JWT in the `Authorization: Bearer <token>` header.
5. NestJS `JwtStrategy` verifies the token using the same `NEXTAUTH_SECRET`.
6. NestJS looks up the email in the `users` whitelist table. If not found → 401.
7. NestJS attaches the user (with role) to the request object.

**Environment variable required on NestJS:**
```
NEXTAUTH_SECRET=<same secret used in nemea-front>
```

**JwtStrategy validates:**
- Token signature (using NEXTAUTH_SECRET)
- Email exists in users table
- Role is extracted from users table (not from token — DB is source of truth for roles)

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| TypeORM | Prisma | Prisma has better type inference and a nicer query API. Use Prisma for new projects starting from scratch with no reference architecture constraints. For Nemea, TypeORM matches the Freedom-Base reference pattern and is already decided. |
| TypeORM | Drizzle | Drizzle is SQL-first and more explicit. Better for teams that prefer raw SQL control. Overkill for this project size. |
| passport-jwt | jose directly | Could verify NextAuth JWT manually without Passport. Valid alternative. Passport provides a more standard NestJS guard integration. |
| @nestjs/config + Joi | Zod for env validation | Zod v4 (already used in frontend) could validate env vars. But @nestjs/config + Joi is the officially documented NestJS pattern. Stick with it for consistency with NestJS docs. |
| NestJS 11 | NestJS 10 | No reason to use v10. v11 is stable, actively maintained, performance improvements. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `synchronize: true` in TypeORM | Drops and recreates columns in production. Data loss risk. Unrecoverable in production. | Migrations always. Even in dev. |
| `@nestjs/class-validator` | 4-year-old unmaintained fork of class-validator. Will break with newer TypeScript or NestJS versions. | `class-validator` (standalone) |
| `@nestjs/class-transformer` | Same — 4-year-old unmaintained fork. | `class-transformer` (standalone) |
| `passport-google-oauth20` on backend | The backend is NOT doing OAuth flows. NextAuth handles all Google OAuth. Adding this creates a parallel auth system with no benefit. | Nothing. Backend only validates JWTs. |
| `ormconfig.json` | Deprecated since TypeORM 0.3. CLI no longer reads it. | `dataSource.ts` with explicit DataSource |
| Storing decimal results from TypeORM without parsing | TypeORM returns DECIMAL as strings. Math operations on strings produce NaN. | `parseFloat()` or column transformer |
| Wildcard CORS (`*`) with credentials | Credentials (JWT headers) cannot be sent with wildcard CORS. Browser blocks it. | Explicit `origin: process.env.FRONTEND_URL` |

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| @nestjs/core ^11.x | @nestjs/typeorm ^11.x | Must match major version. |
| @nestjs/core ^11.x | @nestjs/passport ^11.x | Must match major version. |
| @nestjs/core ^11.x | @nestjs/swagger ^11.x | Must match major version. |
| TypeORM 0.3.x | pg ^8.x | TypeORM 0.3 requires pg v8. pg v7 not supported. |
| passport-jwt ^4.x | @nestjs/passport ^11.x | Use v4. v5 (pending) has breaking changes not yet reflected in NestJS docs. |
| class-validator ^0.14.x | class-transformer ^0.5.x | Must be installed together. They share reflect-metadata. |
| Node.js 20 | NestJS 11 | NestJS 11 requires Node 16+. Node 20 LTS is safe. |

---

## Railway Production Configuration

```
DATABASE_URL=postgresql://USER:PASS@HOST:PORT/DB  (Railway provides this automatically)
NODE_ENV=production
PORT=4000
NEXTAUTH_SECRET=<same as frontend>
FRONTEND_URL=https://nemea-front.vercel.app
```

Railway injects `DATABASE_URL` automatically when you link the PostgreSQL service.
NestJS must enable SSL when `NODE_ENV=production`:
```typescript
ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
```

---

## Sources

- WebSearch: NestJS current version → "11.1.14" confirmed, Trilon announcement Jan 2025 (MEDIUM confidence)
- WebSearch: TypeORM latest version → "0.3.28" confirmed from npm (MEDIUM confidence)
- WebSearch: @nestjs/swagger version → "11.2.6 last published 23 days ago" (HIGH confidence — very recent)
- WebSearch: @nestjs/passport version → "11.0.5" current (MEDIUM confidence)
- WebSearch: NestJS TypeORM migration setup → DataSource pattern confirmed as standard (HIGH confidence — multiple sources agree)
- WebSearch: NextAuth JWT backend validation → jose / NEXTAUTH_SECRET pattern confirmed (MEDIUM confidence)
- WebSearch: TypeORM DECIMAL as string → Known issue confirmed in multiple GitHub issues (HIGH confidence)
- WebSearch: Railway NestJS PostgreSQL → DATABASE_URL + SSL pattern confirmed (HIGH confidence — official Railway docs)
- Training data: Docker Compose postgres:16-alpine, helmet, @nestjs/throttler patterns (MEDIUM confidence — verified against 2025 search results)

---

*Stack research for: NestJS + TypeORM + PostgreSQL backend — Nemea cost management*
*Researched: 2026-02-28*
