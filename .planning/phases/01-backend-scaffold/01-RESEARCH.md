# Phase 1: Backend Scaffold - Research

**Researched:** 2026-02-28
**Domain:** NestJS 11 + TypeORM + PostgreSQL scaffold, Docker Compose, Railway deploy
**Confidence:** HIGH

## Summary

Phase 1 sets up the NestJS 11 backend from scratch inside `nemea-back/`, wiring TypeORM with migrations-only (never synchronize), Docker Compose for local PostgreSQL, SWC for fast hot reload, and all cross-cutting concerns (Swagger, helmet, throttler, env validation, CORS). The project deploys to Railway as a monorepo subdirectory. No business entities are created -- only the foundation that all subsequent phases build on.

The user has made detailed decisions about API format, response envelopes (Claude's discretion), dev setup, route conventions, security, testing, and git hooks. The frontend is already scaffolded with specific ESLint/Prettier config that the backend must replicate for consistency. The critical Windows compatibility issue with `$npm_config_name` in migration scripts must be handled from day 1.

**Primary recommendation:** Use `nest new` with `--strict` flag to scaffold, then configure SWC builder, wire TypeORM with a shared DataSource config (not autoLoadEntities), set up Docker Compose with two PostgreSQL services (dev + test), and ensure Railway deploy works with the monorepo root directory setting.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Error responses follow consistent format: `{statusCode, error, message, timestamp}` via global HttpExceptionFilter
- Validation errors return message as array of individual field errors
- Global ValidationPipe with class-validator + class-transformer for automatic DTO validation
- Swagger UI only in development environment, accessible at `/api/docs`
- Docker Compose obligatorio -- PostgreSQL 16 only (no pgAdmin)
- `.env.example` completo con todos los valores de dev preconfigurados (copiar a .env y funciona)
- SWC compiler para hot reload rapido (`start:dev` usa SWC)
- Linting consistente con frontend: ESLint + Prettier, single quotes, 2 spaces, trailing commas, explicit return types, no `any`
- 4 comandos claros para arrancar: `docker compose up -d` -> `npm install` -> `npm run migration:run` -> `npm run start:dev`
- Puerto 4000 (frontend en 3000)
- Global prefix `/api` en todas las rutas
- Rutas en ingles: `/api/supplies`, `/api/products`, `/api/expenses`
- Sin versionado por ahora (no `/v1/`)
- Health check en `GET /api/health` devuelve `{status, app, version, uptime}`
- Throttler: 100 requests/minuto por IP
- CORS: solo `FRONTEND_URL` del .env (localhost:3000 en dev, dominio Vercel en prod)
- Helmet con config default activado
- Env validation con Joi: la app falla al arrancar si falta una variable requerida (error claro)
- Config Railway (Procfile/nixpacks) incluida desde el dia 1
- Database SSL auto por ambiente: off en dev (Docker local), on en prod (Railway)
- Unit tests (Jest + mocks) + E2E tests (supertest)
- DB separada para E2E: segundo servicio PostgreSQL en Docker Compose (puerto 5433, `nemea_test`)
- Sin threshold de cobertura obligatorio -- reportar cobertura pero no bloquear
- Husky + lint-staged: pre-commit corre lint + prettier check solo sobre archivos en staging
- Tests NO corren en pre-commit (se corren manualmente o en CI)

### Claude's Discretion
- Success response format: envelope `{data, meta}` vs flat NestJS default -- Claude chooses what best fits the project
- Logging level: NestJS default vs request logging completo -- Claude chooses

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| INFR-01 | Backend runs on NestJS 11 with TypeORM and PostgreSQL | Core stack section: NestJS 11.x + @nestjs/typeorm 11.x + TypeORM 0.3.x + pg 8.x. All verified. |
| INFR-02 | Database uses migrations only (never synchronize) | Architecture Patterns: shared DataSource config with `synchronize: false` hardcoded. Migration scripts with Windows-compatible commands. |
| INFR-03 | Docker Compose for local PostgreSQL development | Docker Compose section: PostgreSQL 16-alpine + test DB on port 5433. Health checks included. |
| INFR-04 | All tables have created_at and updated_at timestamps | Base entity pattern with `@CreateDateColumn` and `@UpdateDateColumn`. Implemented in Phase 1 as abstract BaseEntity. |
| INFR-05 | TypeScript strict mode, no any, explicit return types | ESLint flat config section: typescript-eslint with explicit-function-return-type, no-any rules. tsconfig.json with strict: true. |
</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/core | ^11.x | NestJS framework core | Current stable. Released Jan 2025. Performance improvements. |
| @nestjs/common | ^11.x | Common decorators and utilities | Must match @nestjs/core major version |
| @nestjs/platform-express | ^11.x | Express HTTP adapter | Default NestJS platform |
| @nestjs/typeorm | ^11.x | TypeORM integration for NestJS | Official module. Must match NestJS 11 major. |
| typeorm | ^0.3.28 | ORM with migration support | Current stable 0.3.x. DataSource API. |
| pg | ^8.x | PostgreSQL driver | Required peer of TypeORM for Postgres. |
| @nestjs/config | ^3.x | Environment variable management | Official module. isGlobal + Joi validation. |
| joi | ^17.x | Env schema validation | Used with @nestjs/config validationSchema. Fails fast on missing vars. |
| @nestjs/swagger | ^11.x | Swagger/OpenAPI documentation | Official module. Version must match NestJS 11. |
| helmet | ^8.x | HTTP security headers | Express middleware. Default config sufficient. |
| @nestjs/throttler | ^6.x | Rate limiting | Official NestJS module. Global ThrottlerGuard. |
| class-validator | ^0.14.x | DTO validation decorators | Standard standalone package (NOT @nestjs/class-validator). |
| class-transformer | ^0.5.x | DTO serialization | Works with ValidationPipe. Standalone package. |
| reflect-metadata | ^0.2.x | Decorator metadata | Required by NestJS and TypeORM. |
| rxjs | ^7.x | Reactive programming | Required peer of @nestjs/core. |

### Development

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @nestjs/cli | ^11.x | NestJS CLI (scaffold, build, start) | Project generation and dev server |
| @swc/cli | ^0.6.x | SWC compiler CLI | Required for `builder: "swc"` |
| @swc/core | ^1.x | SWC compiler core | Required for SWC builder |
| ts-node | ^10.x | TypeScript execution for CLI | Required by TypeORM CLI to read dataSource.ts |
| tsconfig-paths | ^4.x | Path alias resolution at runtime | Required so TypeORM CLI resolves `@src/*` aliases |
| @nestjs/testing | ^11.x | NestJS test utilities | Unit and E2E test module creation |
| jest | ^29.x | Test runner | NestJS default test runner |
| @types/jest | ^29.x | Jest TypeScript types | Dev dependency |
| supertest | ^7.x | HTTP E2E testing | For E2E tests against live NestJS app |
| @types/supertest | ^6.x | Supertest TypeScript types | Dev dependency |
| eslint | ^9.x | Linter | Flat config (eslint.config.mjs) to match frontend |
| typescript-eslint | ^8.x | ESLint TypeScript plugin | Parser + rules for TS |
| eslint-plugin-sonarjs | ^4.x | Cognitive complexity rules | Matches frontend ESLint config |
| prettier | ^3.x | Code formatter | Must match frontend config exactly |
| husky | ^9.x | Git hooks | Pre-commit hooks for lint-staged |
| lint-staged | latest | Run linters on staged files | Runs ESLint + Prettier on pre-commit |
| typescript | ^5.x | TypeScript compiler | Must match frontend version range |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| TypeORM 0.3.x | Prisma | Better type inference, nicer query API. But TypeORM matches the project architecture reference and is already decided. |
| ts-node for TypeORM CLI | tsx | tsx handles tsconfig paths automatically without `-r tsconfig-paths/register`. Viable alternative, but ts-node is the documented TypeORM pattern. |
| Joi for env validation | Zod | Zod v4 already used in frontend. But @nestjs/config + Joi is the officially documented NestJS pattern. Stick with convention. |
| Jest 29 | Vitest | Faster, native ESM. But NestJS CLI scaffolds Jest and @nestjs/testing is built around it. |

**Installation:**
```bash
# Inside nemea-back/ (after nest new)

# Core (most come with nest new)
npm install @nestjs/typeorm typeorm pg
npm install @nestjs/config joi
npm install class-validator class-transformer
npm install helmet @nestjs/throttler
npm install @nestjs/swagger

# SWC (fast compiler)
npm install -D @swc/cli @swc/core

# Testing
npm install -D @nestjs/testing supertest @types/supertest

# Linting (match frontend)
npm install -D eslint typescript-eslint eslint-plugin-sonarjs prettier

# Git hooks
npm install -D husky lint-staged

# TypeORM CLI support
npm install -D ts-node tsconfig-paths
```

## Architecture Patterns

### Recommended Project Structure (Phase 1 only)

```
nemea-back/
├── docker-compose.yml          # PostgreSQL 16 (dev) + PostgreSQL 16 (test)
├── nest-cli.json               # SWC builder config
├── .env.example                # All dev values pre-filled
├── .env                        # Copied from .env.example (gitignored)
├── .prettierrc.json            # Copied from frontend (minus tailwind plugin)
├── eslint.config.mjs           # Flat config matching frontend conventions
├── .lintstagedrc.json          # lint-staged config for pre-commit
├── .swcrc                      # SWC compiler config (optional, usually not needed)
├── tsconfig.json               # strict: true, path aliases
├── tsconfig.build.json         # Excludes test files
├── Procfile                    # Railway: web: node dist/main.js
├── src/
│   ├── main.ts                 # Bootstrap: CORS, helmet, pipes, swagger, port 4000
│   ├── app.module.ts           # Root module: TypeORM, ConfigModule, ThrottlerModule
│   ├── app.controller.ts       # GET /health endpoint
│   ├── app.service.ts          # Health check logic
│   ├── common/
│   │   ├── entities/
│   │   │   └── base.entity.ts  # Abstract: id, created_at, updated_at (INFR-04)
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts  # Global error format
│   │   ├── interceptors/
│   │   │   └── response.interceptor.ts   # Response envelope (Claude's discretion)
│   │   └── types/
│   │       └── role.enum.ts    # enum Role { ADMIN, USER }
│   ├── config/
│   │   ├── env.validation.ts   # Joi schema for env vars
│   │   └── typeorm.config.ts   # TypeORM config factory for NestJS module
│   └── database/
│       ├── data-source.ts      # Standalone DataSource for TypeORM CLI
│       └── migrations/         # Empty dir (migrations go here in future phases)
│           └── .gitkeep
└── test/
    ├── jest-e2e.json           # E2E Jest config
    ├── app.e2e-spec.ts         # E2E test for health check
    └── setup-e2e.ts            # Test DB connection setup
```

### Pattern 1: Shared DataSource Configuration (CRITICAL)

**What:** Both NestJS runtime and TypeORM CLI use the same entity/migration paths from a single source file.
**When to use:** Always. Prevents the #1 pitfall of two configs falling out of sync.
**Example:**

```typescript
// src/database/data-source.ts -- SINGLE SOURCE OF TRUTH
import { DataSource, DataSourceOptions } from 'typeorm';
import { config } from 'dotenv';
import { join } from 'path';

config(); // Load .env for CLI usage

export const dataSourceOptions: DataSourceOptions = {
  type: 'postgres',
  url: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production'
    ? { rejectUnauthorized: false }
    : false,
  entities: [join(__dirname, '..', '**', '*.entity.{ts,js}')],
  migrations: [join(__dirname, 'migrations', '*.{ts,js}')],
  synchronize: false,   // NEVER true. Hardcoded.
  migrationsRun: true,  // Auto-run migrations on app start
  logging: process.env.NODE_ENV !== 'production',
};

// Used by TypeORM CLI (migration:generate, migration:run)
export const AppDataSource = new DataSource(dataSourceOptions);
```

```typescript
// src/config/typeorm.config.ts -- NestJS module config
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import { dataSourceOptions } from '../database/data-source';

export const typeOrmConfig: TypeOrmModuleOptions = {
  ...dataSourceOptions,
  // Do NOT use autoLoadEntities -- use the shared entities from data-source.ts
};
```

```typescript
// src/app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';
import { typeOrmConfig } from './config/typeorm.config';

@Module({
  imports: [
    TypeOrmModule.forRoot(typeOrmConfig),
    // ...
  ],
})
export class AppModule {}
```

### Pattern 2: SWC Builder in nest-cli.json

**What:** Configure NestJS CLI to use SWC for compilation instead of tsc.
**When to use:** Always in this project (user decision).
**Example:**

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

**Important:** SWC is incompatible with NestJS CLI plugins (like the Swagger plugin that auto-infers ApiProperty from DTOs). This means Swagger decorators must be added manually to DTOs. This is fine -- explicit decorators are better for documentation quality.

### Pattern 3: Global Prefix with Health Check

**What:** Set `/api` as global prefix, health endpoint at `GET /api/health`.
**When to use:** This project (user decision).
**Example:**

```typescript
// src/main.ts
async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix('api');

  // Helmet
  app.use(helmet());

  // CORS
  app.enableCors({
    origin: process.env.FRONTEND_URL,
    credentials: true,
  });

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  // Global exception filter
  app.useGlobalFilters(new HttpExceptionFilter());

  // Swagger (dev only)
  if (process.env.NODE_ENV !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('Nemea API')
      .setDescription('Nemea leather goods cost management API')
      .setVersion('0.1.0')
      .build();
    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('api/docs', app, document);
  }

  const port = process.env.PORT || 4000;
  await app.listen(port);
}
```

### Pattern 4: Abstract BaseEntity for Timestamps (INFR-04)

**What:** All entities extend a base class with `id`, `created_at`, and `updated_at`.
**When to use:** Every entity in the project.
**Example:**

```typescript
// src/common/entities/base.entity.ts
import {
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

export abstract class BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @CreateDateColumn({ name: 'created_at', type: 'timestamptz' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at', type: 'timestamptz' })
  updatedAt: Date;
}
```

### Pattern 5: Env Validation with Joi

**What:** Validate all required env vars at boot time. App crashes with clear error if vars are missing.
**When to use:** Always.
**Example:**

```typescript
// src/config/env.validation.ts
import * as Joi from 'joi';

export const envValidationSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'production', 'test')
    .default('development'),
  PORT: Joi.number().default(4000),
  DATABASE_URL: Joi.string().required(),
  FRONTEND_URL: Joi.string().required(),
});
```

```typescript
// In app.module.ts
ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: envValidationSchema,
  validationOptions: {
    abortEarly: false,  // Show all missing vars at once
  },
}),
```

### Pattern 6: Global HttpExceptionFilter

**What:** Catch all exceptions and return consistent error format.
**Example:**

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Response } from 'express';

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const exceptionResponse =
      exception instanceof HttpException
        ? exception.getResponse()
        : { message: 'Internal server error' };

    const errorResponse = {
      statusCode: status,
      error: HttpStatus[status] || 'Error',
      message:
        typeof exceptionResponse === 'object' && 'message' in exceptionResponse
          ? (exceptionResponse as Record<string, unknown>).message
          : exceptionResponse,
      timestamp: new Date().toISOString(),
    };

    response.status(status).json(errorResponse);
  }
}
```

### Anti-Patterns to Avoid

- **autoLoadEntities: true:** Creates divergence between NestJS runtime and TypeORM CLI. Use shared entities array from data-source.ts instead.
- **synchronize: true in any environment:** Data loss risk. Hardcode false unconditionally.
- **`$npm_config_name` in migration scripts on Windows:** Does not work on MINGW64/Windows. Use positional argument or cross-var package instead.
- **Naming a business module `ConfigModule`:** Clashes with @nestjs/config's ConfigModule. Future phases should use `TiendanubeConfigModule`.
- **Installing `@nestjs/class-validator` or `@nestjs/class-transformer`:** These are 4-year-old unmaintained forks. Use standalone `class-validator` and `class-transformer`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Env var validation | Custom validators | @nestjs/config + Joi validationSchema | Fails fast, validates all vars at once, integrates with NestJS DI |
| Rate limiting | Custom middleware counting requests | @nestjs/throttler ThrottlerGuard | Handles storage, TTL, per-IP tracking, Skip decorator |
| HTTP security headers | Manual header-setting middleware | helmet middleware | Covers 15+ headers (CSP, X-Frame, HSTS, etc.) correctly |
| Error response formatting | Manual try/catch in every controller | Global HttpExceptionFilter + ValidationPipe | Centralized, catches all exceptions including validation errors |
| API documentation | Manual endpoint docs | @nestjs/swagger + DocumentBuilder | Auto-generates from decorators, serves interactive UI |
| Timestamp columns | Manual `new Date()` in every save | @CreateDateColumn + @UpdateDateColumn | TypeORM handles creation and update automatically |
| Migration execution | Manual SQL files | TypeORM migration:generate + migration:run | Generates diff from entity changes, tracks which migrations ran |

## Common Pitfalls

### Pitfall 1: TypeORM CLI Migration Scripts on Windows (MINGW64)

**What goes wrong:** The migration generation script uses `$npm_config_name` to pass the migration name, but this shell variable expansion does not work on Windows (including MINGW64/Git Bash).
**Why it happens:** `$npm_config_name` is a Unix shell feature. Windows CMD and PowerShell don't expand it. MINGW64 has partial support but npm scripts run in CMD.
**How to avoid:** Use one of these Windows-compatible approaches:
```json
{
  "typeorm": "ts-node -r tsconfig-paths/register ./node_modules/typeorm/cli",
  "migration:generate": "npm run typeorm -- migration:generate src/database/migrations/%npm_config_name% -d src/database/data-source",
  "migration:run": "npm run typeorm -- migration:run -d src/database/data-source",
  "migration:revert": "npm run typeorm -- migration:revert -d src/database/data-source"
}
```
Or better -- avoid `npm_config_name` entirely and pass the migration path as a direct argument:
```json
{
  "typeorm": "npx ts-node -r tsconfig-paths/register ./node_modules/typeorm/cli",
  "migration:generate": "npm run typeorm -- migration:generate -d src/database/data-source",
  "migration:run": "npm run typeorm -- migration:run -d src/database/data-source",
  "migration:revert": "npm run typeorm -- migration:revert -d src/database/data-source"
}
```
Usage: `npm run migration:generate -- src/database/migrations/CreateUsersTable`
**Warning signs:** `migration:generate` errors with "missing migration path" or generates file with undefined in the name.

### Pitfall 2: Two TypeORM DataSource Configs Falling Out of Sync

**What goes wrong:** NestJS uses `autoLoadEntities: true` at runtime, but TypeORM CLI needs an explicit entity list. When a new entity is added to a module but not to data-source.ts, `migration:generate` produces empty files.
**Why it happens:** `autoLoadEntities` is NestJS-specific; the CLI cannot use it.
**How to avoid:** Use the shared `dataSourceOptions` from `data-source.ts` in both the CLI DataSource AND `TypeOrmModule.forRoot()`. Never use `autoLoadEntities`.
**Warning signs:** `migration:generate` produces empty migration files after adding a new entity.

### Pitfall 3: SWC Incompatibility with NestJS CLI Plugins

**What goes wrong:** The NestJS Swagger CLI plugin (which auto-infers `@ApiProperty` from DTOs) does not work with the SWC builder. Swagger docs show no properties for DTOs.
**Why it happens:** NestJS CLI plugins are tsc-specific transformers that SWC does not support.
**How to avoid:** Do not enable the Swagger CLI plugin. Add `@ApiProperty()` decorators manually to DTOs. This is actually better practice -- it forces explicit documentation.
**Warning signs:** Swagger UI shows DTO schemas with no properties.

### Pitfall 4: Railway Root Directory vs Config File Path

**What goes wrong:** Railway's "Root Directory" setting changes the build context but the `railway.json`/`railway.toml` config file path does NOT follow the root directory setting. If you put railway.json inside `nemea-back/`, Railway may not find it unless you set the config file path explicitly.
**Why it happens:** Railway treats the config file path as absolute from the repo root.
**How to avoid:** Either (a) use a Procfile inside `nemea-back/` which Railway auto-detects from the root directory, or (b) set the Railway config file path explicitly in the service settings to `/nemea-back/railway.json`.
**Warning signs:** Railway deploy uses wrong start command or fails to find config.

### Pitfall 5: Docker Compose `depends_on` Not Waiting for PostgreSQL Readiness

**What goes wrong:** `depends_on` only waits for the container to start, not for PostgreSQL to be ready to accept connections. The NestJS app may crash on startup with "Connection refused."
**Why it happens:** Container start != process ready. PostgreSQL needs a few seconds to initialize.
**How to avoid:** Add health checks to Docker Compose PostgreSQL services. For the NestJS app running outside Docker (local dev), use TypeORM's built-in retry logic or simply start Docker first, then npm run start:dev (the 4-step startup flow user decided).
**Warning signs:** First `npm run start:dev` after `docker compose up -d` fails with connection error but works on retry.

### Pitfall 6: ESLint Flat Config Transition

**What goes wrong:** NestJS CLI scaffolds a legacy `.eslintrc.js` with `@nestjs/eslint-config`. The frontend uses ESLint 9 flat config (`eslint.config.mjs`). Mixing formats causes confusion.
**Why it happens:** NestJS CLI has not fully migrated to flat config. The scaffolded config is outdated.
**How to avoid:** Delete the scaffolded `.eslintrc.js` and create an `eslint.config.mjs` from scratch matching the frontend pattern. Use `typescript-eslint` and `eslint-plugin-sonarjs` directly.

## Code Examples

### Docker Compose with Dev and Test Databases

```yaml
# docker-compose.yml in nemea-back/
services:
  postgres:
    image: postgres:16-alpine
    container_name: nemea-postgres
    ports:
      - '5432:5432'
    environment:
      POSTGRES_USER: nemea
      POSTGRES_PASSWORD: nemea_dev
      POSTGRES_DB: nemea_db
    volumes:
      - nemea_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U nemea -d nemea_db']
      interval: 5s
      timeout: 5s
      retries: 5

  postgres-test:
    image: postgres:16-alpine
    container_name: nemea-postgres-test
    ports:
      - '5433:5432'
    environment:
      POSTGRES_USER: nemea
      POSTGRES_PASSWORD: nemea_test
      POSTGRES_DB: nemea_test
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U nemea -d nemea_test']
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  nemea_pgdata:
```

### .env.example (Complete, Pre-filled for Dev)

```bash
# App
NODE_ENV=development
PORT=4000

# Database (Docker Compose local)
DATABASE_URL=postgresql://nemea:nemea_dev@localhost:5432/nemea_db

# Database (Test - for E2E tests)
DATABASE_URL_TEST=postgresql://nemea:nemea_test@localhost:5433/nemea_test

# CORS
FRONTEND_URL=http://localhost:3000

# Future: Auth (Phase 2)
# JWT_SECRET=
# NEXTAUTH_SECRET=
```

### ThrottlerModule Global Setup

```typescript
// In app.module.ts imports
ThrottlerModule.forRoot({
  throttlers: [
    {
      ttl: 60000,    // 1 minute in milliseconds
      limit: 100,    // 100 requests per minute per IP
    },
  ],
}),

// In providers
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard,
},
```

### ESLint Flat Config for Backend

```javascript
// eslint.config.mjs (nemea-back/)
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import sonarjs from 'eslint-plugin-sonarjs';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  {
    plugins: { sonarjs },
    rules: {
      'sonarjs/cognitive-complexity': ['error', 30],
      'no-debugger': 'error',
      quotes: [
        'error',
        'single',
        { allowTemplateLiterals: true, avoidEscape: true },
      ],
    },
  },
  {
    files: ['**/*.ts'],
    rules: {
      '@typescript-eslint/no-unused-vars': [
        'error',
        {
          args: 'after-used',
          argsIgnorePattern: '^_',
          varsIgnorePattern: '^_',
          caughtErrors: 'none',
        },
      ],
      '@typescript-eslint/explicit-function-return-type': [
        'error',
        {
          allowTypedFunctionExpressions: true,
          allowHigherOrderFunctions: true,
          allowDirectConstAssertionInArrowFunctions: true,
          allowExpressions: true,
        },
      ],
      '@typescript-eslint/no-explicit-any': 'error',
    },
  },
  {
    ignores: ['node_modules/**', 'dist/**', 'coverage/**'],
  },
);
```

### Prettier Config (Match Frontend)

```json
{
  "endOfLine": "auto",
  "singleQuote": true,
  "trailingComma": "all",
  "tabWidth": 2,
  "semi": true
}
```

Note: Frontend includes `prettier-plugin-tailwindcss` and `jsxSingleQuote` -- backend omits both since it has no JSX/Tailwind.

### lint-staged Config

```json
{
  "*.ts": ["eslint --fix", "prettier --check"],
  "*.json": ["prettier --check"]
}
```

### Husky Pre-commit Hook

```bash
# .husky/pre-commit
npx lint-staged
```

### Procfile for Railway

```
web: node dist/main.js
```

### TypeScript Config (strict mode)

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strict": true,
    "noImplicitAny": true,
    "strictBindCallApply": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@src/*": ["src/*"]
    }
  }
}
```

### Health Check Endpoint

```typescript
// src/app.controller.ts
import { Controller, Get } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';

@ApiTags('health')
@Controller('health')
export class AppController {
  @Get()
  @ApiOperation({ summary: 'Health check' })
  @ApiResponse({ status: 200, description: 'Service is healthy' })
  getHealth(): {
    status: string;
    app: string;
    version: string;
    uptime: number;
  } {
    return {
      status: 'ok',
      app: 'nemea-back',
      version: '0.1.0',
      uptime: process.uptime(),
    };
  }
}
```

## Claude's Discretion: Recommendations

### Response Envelope: Use `{data, meta}` Envelope

**Recommendation:** Wrap successful responses in `{data, meta}` envelope via a global interceptor.

**Rationale:**
- Consistent shape for all responses (errors already have a fixed format)
- `meta` can hold pagination info in future phases (product list, supply list)
- Frontend can rely on a single response interface: `{ data: T, meta?: { total, page, limit } }`
- Easy to implement with a global `ResponseInterceptor`

```typescript
// Success response shape
{
  data: T,
  meta?: {
    total?: number;
    page?: number;
    limit?: number;
  }
}
```

### Logging: Use NestJS Default Logger + Request Logging Interceptor

**Recommendation:** Use NestJS built-in logger (not Winston or Pino) with a lightweight `LoggingInterceptor` that logs method, URL, status code, and response time for every request.

**Rationale:**
- NestJS default logger is sufficient for 2-3 users
- Railway captures stdout automatically (no external logging service needed)
- A `LoggingInterceptor` adds request/response visibility without a heavy logging framework
- Can upgrade to Pino if performance becomes a concern (it won't at this scale)

```typescript
// Example log output:
// [Nest] GET /api/health 200 3ms
// [Nest] POST /api/supplies 201 45ms
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `.eslintrc.js` (NestJS default) | `eslint.config.mjs` flat config | ESLint 9.x (2024) | Must delete scaffolded config and create flat config |
| `ormconfig.json` | `data-source.ts` DataSource export | TypeORM 0.3 (2022) | CLI reads TS DataSource file directly |
| `autoLoadEntities: true` | Shared entities array from data-source.ts | Best practice since TypeORM 0.3 | Prevents CLI/runtime divergence |
| Nixpacks (Railway builder) | Railpack (new default) | Feb 2026 | New Railway services default to Railpack. Nixpacks still works. Both detect Node.js automatically. |
| NestJS 10 | NestJS 11 | Jan 2025 | Improved module startup performance, structured logging |

**Deprecated/outdated:**
- `ormconfig.json`: Deprecated since TypeORM 0.3. CLI ignores it.
- `@nestjs/class-validator` / `@nestjs/class-transformer`: Unmaintained 4-year-old forks. Use standalone packages.
- `.eslintrc.js`: Legacy format. ESLint 9 flat config is current.
- Nixpacks: Deprecated by Railway, replaced by Railpack. Existing services still work.

## Open Questions

1. **Railway builder: Nixpacks vs Railpack**
   - What we know: Nixpacks is deprecated, Railpack is the new default. Both auto-detect Node.js. NestJS Railway guide still references Nixpacks.
   - What's unclear: Whether Railpack handles Procfile identically to Nixpacks. Whether monorepo root directory + Railpack works the same.
   - Recommendation: Use a `Procfile` (universally supported by both builders). If Railpack causes issues, Railway settings can switch the builder. This is a low-risk question.

2. **Path alias in TypeORM CLI: `@src/*` resolution**
   - What we know: tsconfig-paths resolves aliases at runtime. TypeORM CLI uses ts-node which supports `-r tsconfig-paths/register`.
   - What's unclear: Whether path aliases in entity imports cause issues during `migration:generate` on Windows.
   - Recommendation: Use path aliases sparingly in entity files. If migration generation fails, use relative imports in entity files as fallback. Test this early in Plan 01-02.

3. **nest new vs manual scaffold**
   - What we know: `nest new` scaffolds a working project with Jest, tsc, and basic module structure. But it generates legacy ESLint config and does not include SWC.
   - What's unclear: Whether the time saved by `nest new` is worth the cleanup of deleting/replacing generated configs.
   - Recommendation: Use `nest new nemea-back --strict --package-manager npm` to get the correct NestJS boilerplate, then replace ESLint config, add SWC, and customize. Faster than manual scaffold.

## Sources

### Primary (HIGH confidence)
- NestJS official SWC docs: https://docs.nestjs.com/recipes/swc -- SWC builder config in nest-cli.json, CLI plugin incompatibility
- NestJS official docs: https://docs.nestjs.com/techniques/configuration -- @nestjs/config + Joi validationSchema pattern
- NestJS official docs: https://docs.nestjs.com/security/rate-limiting -- ThrottlerModule forRoot configuration
- NestJS official docs: https://docs.nestjs.com/exception-filters -- Global exception filter pattern
- Railway monorepo guide: https://docs.railway.com/guides/monorepo -- Root directory, config file path behavior
- Railway NestJS guide: https://docs.railway.com/guides/nest -- Deployment methods, Dockerfile pattern
- Railway build config: https://docs.railway.com/builds/build-configuration -- Railpack vs Nixpacks status

### Secondary (MEDIUM confidence)
- Multiple Medium/DEV articles on TypeORM migration setup with ts-node and tsconfig-paths
- Multiple articles on ESLint 9 flat config with typescript-eslint
- NestJS Swagger setup with conditional production disable: https://github.com/nestjs/swagger/issues/184
- Docker Compose PostgreSQL health check patterns from community guides

### Tertiary (LOW confidence)
- `$npm_config_name` Windows incompatibility: inferred from general npm behavior on Windows + community reports. Test in Plan 01-02 to verify.
- Railpack Procfile handling: no official NestJS-specific documentation yet for Railpack. Using Procfile as safe universal approach.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- versions verified via npm, NestJS 11 is current stable
- Architecture: HIGH -- patterns from NestJS official docs + verified pitfall research
- Pitfalls: HIGH -- validated against existing project research (PITFALLS.md) + Windows-specific findings
- Railway deploy: MEDIUM -- Railpack transition is recent, monorepo setup needs runtime verification
- Windows migration scripts: MEDIUM -- needs testing during implementation

**Research date:** 2026-02-28
**Valid until:** 2026-03-30 (30 days -- stable stack, no fast-moving dependencies)
