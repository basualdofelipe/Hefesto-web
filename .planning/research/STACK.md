# Technology Stack: v1.1 Additions

**Project:** Nemea v1.1 -- Tiendanube Pricing, Investor Dashboard, Hardening
**Researched:** 2026-03-26
**Scope:** NEW libraries and patterns only. Existing stack (NestJS 11, Next.js 16, TypeORM, etc.) is validated and not re-researched.

---

## Executive Summary

**v1.1 requires zero new npm dependencies.** The existing stack covers every feature. The calculadora migration, scenario simulator, investor dashboard, and hardening work are all achievable with current libraries. The key additions are architectural (new NestJS modules, new entities, proxy.ts wiring) not library additions.

The one area that deserves explicit justification is **decimal precision for financial calculations** -- the recommendation is to use native JavaScript `Number` (IEEE 754 double) with disciplined rounding, matching the existing prototype and the current CostsService pattern. A precision library like `decimal.js` is unnecessary for this domain.

---

## New Stack Components: None Required

### Why No New Libraries

| v1.1 Feature | Implementation Approach | Libraries Needed |
|---|---|---|
| Config Tiendanube (admin-editable rates) | New TypeORM entities + NestJS CRUD module | Already have: TypeORM, class-validator, class-transformer |
| Calculadora forward/inverse | New NestJS service with pure math functions | Native TypeScript -- no external math library |
| Investor dashboard | New frontend page reading existing `/products` endpoint with costs | Already have: React, Shadcn/ui tables, Zod |
| Scenario simulator | New TypeORM entity (`pricing_scenario`) + CRUD | Already have: TypeORM, react-hook-form, Zod |
| Product hierarchical grouping | Frontend-only collapsible table UI | Already have: Radix UI accordion/collapsible, Lucide icons |
| Proxy.ts wiring | Rename export + move file position | Already have: `proxy.ts` with `auth()` wrapper |
| 401 handling | Add redirect logic in `apiClientFetch` | Already have: `next-auth` session, `signOut` |
| DRY cleanup | Consolidate duplicated interfaces/utils | No new code -- refactoring only |

---

## Financial Calculation Precision: Native Number

### Decision: Use `Number` with `Math.round()`, NOT decimal.js

**Confidence:** HIGH

**Rationale:**

1. **Domain fit.** Argentine pesos are quoted to 2 decimal places. Rates are percentages with 2 decimal places. The prototype (`calculadora-tiendanube.jsx`) uses plain JavaScript numbers with no issues. The existing `CostsService` already uses `parseFloat()` and `Math.round(x * 100) / 100` -- same pattern.

2. **Precision analysis.** IEEE 754 double-precision gives 15-17 significant digits. The largest realistic calculation: `$10,000,000 * 37.39% = $3,739,000`. This is well within safe integer range (~9 quadrillion). Rounding errors only appear at the 16th digit -- irrelevant for 2-decimal ARS amounts.

3. **The prototype works.** The calcForward/calcInverse functions in the JSX prototype use native numbers and produce correct results against real business data (validated test case: Billetera Hefesto at $87,000). Introducing `decimal.js` would mean rewriting working logic for zero benefit.

4. **Consistency.** The existing `CostsService.buildCostMap()` uses `Math.round(unitPrice * quantity * 100) / 100`. The calculadora service should follow the same convention.

5. **Bundle cost avoided.** `decimal.js` is 32KB minified. For a backend-only calculation, bundle size is irrelevant -- but the cognitive overhead of a new API (`new Decimal(x).mul(y).toNumber()`) adds complexity with no return.

**The rule:** All monetary results get `Math.round(value * 100) / 100` before being stored or returned. All percentage rates are stored as numbers (e.g., `7.69` not `0.0769`). This matches the prototype and the existing codebase.

### When decimal.js WOULD be justified (not this project)

- Calculations involving more than 15 significant digits (crypto, scientific)
- Currency conversions with 6+ decimal place exchange rates
- Accumulation of thousands of rounding operations in a single pipeline
- Regulatory requirement for exact decimal arithmetic (banking)

None of these apply to Nemea.

---

## Backend: New NestJS Modules

### TiendanubeModule (NEW)

Replaces the previously planned `ConfigModule` (business config). Already correctly scoped in the architecture research to avoid naming clash with `@nestjs/config`.

| Component | Purpose | Pattern |
|---|---|---|
| `TiendanubeModule` | Registers entities, exports service | Standard NestJS module |
| `TiendanubeConfigService` | CRUD for plan rates, installment rates, tax config | Standard CRUD service |
| `TiendanubeConfigController` | `GET/PUT /tiendanube/config` (ADMIN only) | `@Roles('ADMIN')` protected |
| `CalculadoraService` | `calcForward()` + `calcInverse()` pure functions | Stateless service, no DB access |
| `CalculadoraController` | `POST /tiendanube/calculate` (any role) | Accepts body with pricing params, returns breakdown |
| Entities: `TiendanubePlan`, `InstallmentRate`, `TiendanubeSettings` | Config tables | TypeORM entities extending `BaseEntity` |

**Key design choice:** `CalculadoraService` is a pure calculation service with zero DB dependencies. It receives all config as parameters. The controller orchestrates: reads config from `TiendanubeConfigService`, passes it to `CalculadoraService`. This makes the calculator testable without a database.

### ScenariosModule (NEW)

| Component | Purpose | Pattern |
|---|---|---|
| `ScenariosModule` | Registers entities, exports service | Standard NestJS module |
| `ScenariosService` | CRUD for pricing scenarios per user | Standard CRUD with user ownership |
| `ScenariosController` | `GET/POST/PUT/DELETE /scenarios` | User-scoped (each user sees only their scenarios) |
| Entity: `PricingScenario` | Stores scenario name, user FK, created_at | TypeORM entity extending `BaseEntity` |
| Entity: `ScenarioProduct` | Stores product FK, overridden sale price, scenario FK | Join table with price override |

**Key design choice:** Scenarios store only overrides (product_id + override_price). The dashboard reads real costs from `CostsService` and applies overrides in-memory. No data duplication.

### UsersModule (UPDATE -- admin UI)

The backend CRUD already exists. The frontend needs a new page at `/(app)/usuarios/page.tsx`. No backend changes needed.

---

## Frontend: New Patterns

### proxy.ts Wiring (HARDENING)

**Current state:** `proxy.ts` exists at `nemea-front/src/proxy.ts` with the correct `auth()` wrapper and `proxy` export. It handles unauthenticated redirects and missing `accessToken` redirects.

**Problem identified in PROJECT.md:** The proxy is not wired as `middleware.ts` -- it exists as `proxy.ts` but may not be in the correct location for Next.js 16 to pick it up.

**Verification needed at implementation time:**
- `proxy.ts` must be at the root of `src/` (or project root if no `src/`). Currently at `nemea-front/src/proxy.ts` -- this is the correct location for Next.js 16 with `src/` directory.
- The export must be named `proxy` (not `middleware`). Currently: `export const proxy = auth(...)` -- correct.
- The `config` export with `matcher` must be present. Currently present.

**Conclusion:** The proxy file is already correctly structured for Next.js 16. The issue noted in PROJECT.md ("proxy.ts no wired as middleware.ts") may be stale. The hardening task should verify this works in production, not rewrite it.

**Confidence:** MEDIUM -- needs runtime verification. The file structure looks correct but the PROJECT.md note suggests it may not actually be intercepting requests.

### 401 Handling in apiClientFetch (HARDENING)

**Current state:** `apiClientFetch` in `nemea-front/src/lib/api-client.ts` throws a generic `Error` on non-OK responses. No special handling for 401 Unauthorized (JWT expiry).

**Required change:** On 401 response, trigger `signOut()` from `next-auth` to clear the stale session and redirect to login. This is a ~5 line change in the existing file:

```typescript
// Pattern to implement (no new library needed):
if (res.status === 401) {
  // Import signOut at top of file
  await signOut({ redirectTo: '/login' });
  throw new Error('Session expired');
}
```

**Caveat:** `signOut` from `next-auth/react` is a client-side function. The `apiClientFetch` is already `'use client'`. This works without server-side workarounds.

For `apiFetch` (server-side variant in `api.ts`), 401 handling is different -- it should throw a specific error that the calling Server Component catches and redirects via `redirect('/login')` from `next/navigation`.

### DRY Cleanup Targets (HARDENING)

Per PROJECT.md: `SupplyOption 8x`, `UNIT_LABELS 5x`, `formatDate 4x`. These are pure refactoring -- extract to shared files under `nemea-front/src/lib/` or `nemea-front/src/constants/`. No new libraries needed.

### Product Hierarchical Grouping (FRONTEND UX)

**Current approach:** Products displayed in a flat table. Grouping by type > name > finish is frontend-only.

**Implementation:** Collapsible row groups using Radix UI Collapsible or Accordion (already available via `radix-ui` package). The grouping logic runs client-side on the already-fetched products array. No backend changes.

**Alternative considered:** Server-side grouping via a new endpoint. Rejected because the product count is small (<100), the grouping is a display concern, and the current GET /products already returns all the catalog dimensions needed for grouping.

### Investor Dashboard Page (NEW PAGE)

New page at `/(app)/dashboard/page.tsx`. Reads from existing endpoints:
- `GET /products` (with costs enriched by CostsService)
- `GET /tiendanube/config` (new -- to show current plan rates)
- `GET /scenarios/:id` (new -- to load saved scenario overrides)

**UI components needed:** All available in Shadcn/ui:
- `Table` for catalog margin summary
- `Card` for KPI summaries (total cost, average margin)
- `Select` for scenario picker
- `Input` for inline price override editing
- `Button` for save scenario

No chart library needed for v1.1. The dashboard is a table with calculated columns, not a visualization dashboard.

### Calculadora Page (NEW PAGE)

New page at `/(app)/tiendanube/calculadora/page.tsx`. The UI is a form (react-hook-form + Zod for inputs) that calls `POST /tiendanube/calculate` and displays results.

Key difference from the standalone prototype: the `costoProducto` field can be auto-populated by selecting a product from the DB (costs come from CostsService). The user can still override manually.

No new UI libraries. The prototype's inline styles become Tailwind + Shadcn/ui components.

---

## Database: New Entities

All new tables use the existing `BaseEntity` pattern (UUID PK, `created_at`, `updated_at`).

### Tiendanube Config Tables

| Table | Columns | Notes |
|---|---|---|
| `tiendanube_plans` | `id`, `slug` (unique), `label`, `card_rate_1d`, `card_rate_7d`, `card_rate_14d`, `transfer_rate`, `tx_tiendanube_rate`, `is_active` | One row per plan. Rates as `decimal(5,2)`. |
| `tiendanube_installment_rates` | `id`, `installments` (int, unique), `rate` | Cuotas sin interes rates. Rate as `decimal(5,2)`. |
| `tiendanube_settings` | `id`, `iva_rate`, `iibb_rate` | Single-row config. Rates as `decimal(5,2)`. |

**Design choice:** Separate tables instead of a single JSON blob. Reasons:
1. Each plan row is independently editable without overwriting others.
2. Installment rates can be added/removed (e.g., 18 cuotas) without schema changes.
3. Settings is a single-row table for global tax config.
4. TypeORM entities with `decimal` columns match the existing pattern (see `Expense.amount`).

**Alternative rejected:** Storing all config as JSONB in a single row. Rejected because it loses type safety at the DB level, makes partial updates harder, and doesn't benefit from TypeORM's column validation.

### Scenario Tables

| Table | Columns | Notes |
|---|---|---|
| `pricing_scenarios` | `id`, `user_id` (FK to users), `name`, `description` (nullable) | User-owned scenario definition. |
| `scenario_products` | `id`, `scenario_id` (FK), `product_id` (FK), `sale_price` decimal(12,2) | Price override per product in a scenario. |

**Design choice:** Scenarios are minimal -- just a name and a set of (product, price) overrides. The margin calculation happens at read time by combining:
1. Real cost from `CostsService.calculateAll()`
2. Override price from `scenario_products`
3. Tiendanube config from `tiendanube_plans` + `tiendanube_installment_rates`

This avoids storing derived data and ensures margins always reflect current costs.

---

## What NOT to Add

| Library/Pattern | Why NOT | What to Do Instead |
|---|---|---|
| `decimal.js` / `big.js` / `dinero.js` | Overkill for ARS 2-decimal calculations. Adds API complexity for zero precision benefit at this scale. | Native `Number` + `Math.round(x * 100) / 100` |
| `recharts` / `chart.js` / any charting library | v1.1 dashboard is a table with calculated columns, not charts. Charts are v2+ if ever. | Shadcn `Table` component |
| `@tanstack/react-table` | Product table is <100 rows with simple grouping. TanStack Table's API is heavy for this use case. | Manual grouping with `Array.reduce()` + Radix Collapsible |
| `redis` / `@nestjs/cache-manager` | No caching layer needed. 2-3 users, <100 products, on-the-fly calculation is fast enough. | Direct DB queries via TypeORM |
| `@nestjs/bull` / `@nestjs/schedule` | No background jobs needed. All calculations are synchronous and fast. | Direct service calls |
| `class-transformer` `@Expose`/`@Exclude` on responses | Already using manual DTO mapping in services. Don't introduce serialization decorators mid-project. | Continue with manual DTO mapping |
| A separate "pricing" microservice | Single monolithic NestJS app serves 2-3 users. Microservices add deployment complexity for zero benefit. | New module within existing NestJS app |
| `next-intl` / i18n libraries | App is internal, Spanish-only, 2-3 users. Hardcoded Spanish strings are fine. | Inline Spanish strings |

---

## Integration Points for New Features

### CalculadoraService <-> CostsService

The calculadora needs product costs from `CostsService` to auto-populate `costoProducto`. Two integration approaches:

**Recommended:** `CalculadoraController` calls `CostsService.calculateForProduct(productId)` to get cost, then passes it to `CalculadoraService.calcForward()`. The services don't import each other -- the controller orchestrates.

**Why:** Keeps `CalculadoraService` pure (testable without DB). Follows the existing pattern where `ProductsController` orchestrates between `ProductsService` and `CostsService`.

### ScenariosModule <-> CostsModule + TiendanubeModule

The scenario dashboard read flow:
1. `ScenariosController` loads scenario (price overrides) from DB
2. `CostsService.calculateAll()` provides real costs per product
3. For each product: if scenario has override, use override price; else use product's current sale price
4. `CalculadoraService.calcForward()` computes margin with Tiendanube config

All orchestration happens in the controller or a dedicated `ScenarioDashboardService`. No circular dependencies.

### Proxy.ts <-> NextAuth Session

Already wired. The `auth()` wrapper in `proxy.ts` accesses `req.auth` (the NextAuth session). No changes to the proxy pattern needed -- just verification that it intercepts all routes correctly.

---

## Version Compatibility Check

All existing packages are compatible with the new features. No version bumps required.

| Existing Package | Current Version | Compatible with v1.1? | Notes |
|---|---|---|---|
| `@nestjs/common` | ^11.0.1 | YES | New modules use standard NestJS patterns |
| `typeorm` | ^0.3.28 | YES | New entities follow existing BaseEntity pattern |
| `class-validator` | ^0.14.4 | YES | New DTOs use same decorators |
| `zod` | ^4.3.6 | YES | New frontend forms use same Zod schemas |
| `react-hook-form` | ^7.71.2 | YES | Calculadora form uses same pattern |
| `radix-ui` | ^1.4.3 | YES | Collapsible/Accordion for product grouping |
| `next-auth` | ^5.0.0-beta.30 | YES | signOut for 401 handling already available |
| `next` | 16.1.6 | YES | proxy.ts already correct for Next.js 16 |

---

## Migration Path: Calculadora JSX -> NestJS Service

The existing prototype at `_archive/calculadora/calculadora-tiendanube.jsx` contains two functions that migrate directly:

### calcForward -- Direct Translation

```typescript
// calculadora-tiendanube.jsx line 156 -> TiendanubeCalculadoraService
// Input: all params as typed interface (not loose function args)
// Output: typed CalculationResult interface (not anonymous object)
// Math: identical -- native Number with same formulas
// Change: rates come from DB (TiendanubeConfigService) not hardcoded PLANS object
```

### calcInverse -- Binary Search Migration

```typescript
// calculadora-tiendanube.jsx line 213 -> TiendanubeCalculadoraService
// Algorithm: binary search, 100 iterations (same as prototype)
// Change: none to the algorithm. Wraps calcForward with same convergence logic.
// Note: Could be replaced with algebraic inverse in future, but binary search
//       works correctly and is the proven approach from the prototype.
```

**Testing:** The business rules document provides exact test cases (Billetera Hefesto at $87,000). The migrated service MUST reproduce the same numbers as the prototype for these inputs.

---

## Sources

- [Next.js 16 proxy.ts documentation](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) -- HIGH confidence (official docs, fetched 2026-03-26)
- [Next.js 16 middleware to proxy migration](https://nextjs.org/docs/messages/middleware-to-proxy) -- HIGH confidence (official docs)
- [Auth.js v5 session protection patterns](https://authjs.dev/getting-started/session-management/protecting) -- MEDIUM confidence (official docs via WebSearch)
- [decimal.js npm package](https://www.npmjs.com/package/decimal.js) -- HIGH confidence (npm registry)
- [Decimal.js vs BigNumber.js comparison](https://medium.com/@josephgathumbi/decimal-js-vs-c1471b362181) -- MEDIUM confidence (WebSearch, single source)
- [TypeORM decimal handling with NestJS](https://www.samundra.com.np/creating-a-field-type-to-store-amount-using-typeorm-and-nestjs/1726) -- MEDIUM confidence (WebSearch, aligns with existing codebase pattern)
- [TypeORM DECIMAL as string quirk](https://medium.com/@matthew.bajorek/how-to-properly-handle-decimals-with-typeorm-f0eb2b79ca9c) -- HIGH confidence (matches existing codebase behavior in CostsService)
- Existing codebase analysis: `CostsService`, `proxy.ts`, `apiClientFetch`, `auth.ts`, `calculadora-tiendanube.jsx` -- HIGH confidence (primary source)

---

*Stack research for: Nemea v1.1 Tiendanube & Investor Dashboard*
*Researched: 2026-03-26*
