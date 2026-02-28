# Architecture

**Analysis Date:** 2026-02-28

## Pattern Overview

**Overall:** Monorepo with independent micro-frontends/microservice

**Key Characteristics:**
- Two completely independent sub-projects with separate repos, package managers, and deployment pipelines
- Frontend uses Next.js 16 with App Router (server-side rendering + client components) and Shadcn/ui design system
- Backend uses NestJS with modular architecture (by domain) and TypeORM for data access
- Shared patterns: strict TypeScript, explicit return types, single quotes, no `any`
- Both projects have identical code quality tooling: ESLint + Prettier + Jest from day 1

## Layers

**Frontend (nemea-front/):**
- Purpose: Web UI for product/supply management, pricing calculator, and investor dashboard
- Location: `nemea-front/src/`
- Contains: Next.js pages, React components, custom hooks, type definitions, utilities
- Depends on: React 19, Next.js 16, Tailwind CSS, Shadcn/ui (Radix primitives), next-themes, react-hook-form, Zod for validation
- Used by: Browser clients (desktop, potentially mobile via responsive design)

**Backend (nemea-back/):**
- Purpose: REST API providing CRUD operations, business logic, authentication, and cost calculation
- Location: `nemea-back/src/` (currently scaffolded with no actual modules yet)
- Contains: NestJS controllers, services, modules, TypeORM entities, migrations
- Depends on: NestJS, TypeORM, PostgreSQL (Railway), JWT for auth validation
- Used by: Frontend via HTTP requests, external Tiendanube webhooks (future)

## Data Flow

**User Authentication Flow:**

1. User visits `/login` on frontend
2. NextAuth initiates Google OAuth flow
3. Google redirects back with authorization code
4. Frontend stores NextAuth JWT token
5. Frontend makes request to backend with JWT in Authorization header
6. Backend validates JWT signature and checks user email whitelist in `users` table
7. Whitelist check determines role (ADMIN or USER)
8. Response includes role-based data visibility

**Product Cost Calculation Flow:**

1. Frontend requests `/products` from backend
2. Backend CostsModule calculates cost per product:
   a. Fetch product composition from `supplies_per_product_history` (latest)
   b. Fetch latest price for each supply from `supplies_price_history`
   c. Multiply quantity × price for each material
   d. Sum all material costs
3. Backend returns products with calculated costs
4. Frontend displays in read-only view (USER) or editable table (ADMIN)

**Tiendanube Calculator Flow (Phase 6):**

1. Frontend requests current `config_tiendanube` from backend
2. User inputs desired profit margin or retail price
3. Calculator uses config (card rates, transfer fee, IIBB, IVA) to compute:
   - Forward: cost → suggested retail price
   - Inverse: desired price → implied profit
4. All rates (commission, IIBB, IVA) are editable by ADMIN in `/configuracion`

**State Management:**

- Frontend: React Context for theme (next-themes), form state via react-hook-form, server state via HTTP requests to backend
- Backend: Immutable request/response DTO pattern, no in-memory caching (Phase 5 decision pending on caching strategy)
- Database: Single source of truth in PostgreSQL (Railway), no caching layer yet

## Key Abstractions

**Frontend Components (Shadcn/ui system):**
- Purpose: Reusable, accessible UI primitives with Radix UI + Tailwind CSS
- Examples: `src/components/ui/button.tsx`, `src/components/ui/input.tsx`, `src/components/ui/card.tsx`
- Pattern: Each component is a function that composes Radix primitive + CVA (class-variance-authority) for styling + cn() utility for merging Tailwind classes. No barrel files yet (`src/components/ui/` is flat).

**Frontend Providers:**
- Purpose: Inject global state/context into React tree
- Examples: `src/providers/ThemeProvider.tsx` wraps next-themes ThemeProvider
- Pattern: Named export factory functions that accept children ReactNode, wrapped in RootLayout

**Frontend Utilities:**
- Purpose: Shared helper functions used across pages/components
- Examples: `src/lib/utils.ts` exports `cn()` for Tailwind class merging
- Pattern: Pure functions, no side effects, explicit return types

**Backend Modules (NestJS):**
- Purpose: Domain-driven organization of business logic
- Planned modules (from scope-v1.md):
  - AuthModule: Google OAuth validation, JWT, whitelist enforcement
  - SuppliesModule: CRUD supplies, suppliers, supply types, price history
  - ProductsModule: CRUD products, catalogs (type/name/finish/color/size)
  - CostsModule: Cost calculation logic (fetch supplies, sum × price)
  - ConfigModule: CRUD Tiendanube config (rates, cuotas, IIBB, IVA)
- Pattern: NestJS standard — controller (HTTP layer), service (business logic), entity (data), repository (data access via TypeORM)

**DTOs (Data Transfer Objects):**
- Purpose: Define contract between frontend and backend, validate request/response shape
- Pattern: Class-based with decorators (class-validator) for validation
- Used by: NestJS controllers to validate request body, frontend to type API responses

**TypeORM Entities:**
- Purpose: Define database schema and relationships
- Pattern: Class-based with decorators (`@Entity`, `@Column`, `@ManyToOne`, etc.)
- Migrations: Always generated from entities, never synchronize:true (professional practice from day 1)

## Entry Points

**Frontend (nemea-front/):**
- Location: `src/app/layout.tsx` (server component) → `src/app/page.tsx` (client component)
- Triggers: Browser request to `/`
- Responsibilities:
  - Render HTML shell with metadata, fonts (Engraving, Poppins, JetBrains Mono)
  - Wrap React tree with providers (ThemeProvider)
  - Mount Sonner toast notification system

**Backend (nemea-back/):**
- Location: `src/main.ts` (scaffolded, implementation pending Phase 2)
- Triggers: Railway server startup
- Responsibilities:
  - Initialize NestJS app
  - Enable CORS for frontend domain (Vercel)
  - Configure helmet for security headers
  - Load environment config from .env
  - Start listening on port 4000

## Error Handling

**Strategy:** Explicit error responses, no silent failures

**Frontend:**
- Patterns:
  - Try/catch in async functions with toast notification on error
  - Form validation via react-hook-form + Zod schema before submit
  - 404/500 pages in `src/app/` if implemented
  - Mock pattern for dependencies: see `src/app/__tests__/page.test.tsx` using jest.mock()

**Backend (planned for Phase 2+):**
- Patterns:
  - NestJS built-in exception filters for HTTP errors
  - Validation via class-validator decorators on DTO classes
  - Structured error response: `{ statusCode, message, error }`
  - No console.error in production (next.config.mjs removes non-error console logs)

## Cross-Cutting Concerns

**Logging:**
- Frontend: none yet (console is stripped in production)
- Backend: planned via Winston logger in Phase 2

**Validation:**
- Frontend: react-hook-form + Zod schemas for forms
- Backend: class-validator decorators on DTOs

**Authentication:**
- Frontend: NextAuth.js handles OAuth flow, stores JWT in secure HTTP-only cookie
- Backend: JWT validation middleware, whitelist check against `users` table
- Authorization: role-based guards (ADMIN vs USER) on endpoints

**Styling:**
- Frontend: Tailwind CSS v4 + CSS variables for theme (dark/light)
- Design tokens: Shadcn/ui new-york preset with neutral base color
- Custom fonts: Engraving CC (display), Poppins (body), JetBrains Mono (code)

**Type Safety:**
- Frontend: TypeScript strict mode, explicit return types on all functions (enforced via ESLint)
- Backend: TypeScript strict mode (to be set up in Phase 2)
- Exception: Shadcn/ui components exempt from explicit return type rule (see eslint.config.mjs)

---

*Architecture analysis: 2026-02-28*
