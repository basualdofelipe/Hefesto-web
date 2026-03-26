# Domain Pitfalls: v1.1 Tiendanube & Investor Dashboard

**Domain:** Financial calculation services, investor dashboard, scenario simulator added to existing NestJS + Next.js leather goods cost management app
**Researched:** 2026-03-26
**Confidence:** HIGH (grounded in actual codebase analysis, official docs, and v1 senior review findings)

---

## Critical Pitfalls

Mistakes that cause rewrites, data corruption, or silent financial errors.

### Pitfall 1: Floating-Point Arithmetic in Tiendanube Commission Calculations

**What goes wrong:**
The Tiendanube calculator computes chains of percentages on monetary values: commission rate + IVA, installment fees, IIBB retention, IVA credit/debit. JavaScript's `Number` type is IEEE 754 double-precision float. Chaining multiplications and divisions accumulates error. Example from the existing calculator prototype:

```javascript
// calcForward in calculadora-tiendanube.jsx
const tasaConIVA = tasaBase * (1 + IVA_RATE);          // 7.69 * 1.21 = 9.3049 (OK here)
const comisionPagoNube = totalCliente * (tasaConIVA / 100); // fine at this scale
const ivaCreditoComision = comisionPagoNube * (IVA_RATE / (1 + IVA_RATE)); // compound error
```

At `precioVenta = $87,000` the error is sub-centavo (invisible). But at `precioVenta = $500,000` (batch B2B scenario) with 12 installments, the accumulated error across 7 intermediate calculations can reach $1-3 ARS -- visible in a "ganancia neta" comparison between scenarios.

**Why it happens:**
The prototype was a standalone React component using `parseFloat()` everywhere and `Math.round(x * 100) / 100` as the only rounding. The existing backend `CostsService` already does the same pattern: `Math.round(unitPrice * quantity * 100) / 100`. This works for simple cost = price x quantity, but the Tiendanube formula has 7 dependent intermediate steps where errors compound.

**Consequences:**
- Two scenarios with different payment methods show "identical" net profit that differs by $1-2 when compared in the dashboard
- Inverse calculator (binary search) converges to wrong price because it targets an imprecise forward result
- Users lose trust in the calculator when the numbers don't match their accountant's spreadsheet

**Prevention:**
- Use integer arithmetic in centavos (multiply all monetary inputs by 100, work in integers, divide by 100 only for display). This is the simplest approach for ARS which has exactly 2 decimal places.
- Alternative: use `Decimal.js` for the calculation service. The backend already stores `decimal(12,2)` in PostgreSQL as strings via TypeORM (the `Expense.amount` and `SupplyPriceHistory.price` columns are typed as `string` in entities). Parse to Decimal, calculate, store back.
- Round only ONCE at the end of the full chain, not at intermediate steps. The current `Math.round(x * 100) / 100` at line 110 of `costs.service.ts` is doing intermediate rounding per line item.
- For the inverse calculator: the binary search needs an epsilon comparison (`Math.abs(result - target) < 0.01`) not an iteration count. The current prototype runs exactly 100 iterations blindly.

**Detection:**
- Test: calculate forward with precio $87,000, then calculate inverse targeting the resulting ganancia. The inverse result should round-trip back to $87,000 +/- $0.01. If it differs by more, floating point is the cause.

**Phase to address:** Tiendanube calculation service phase. Design the number representation BEFORE writing any formula code.

---

### Pitfall 2: proxy.ts Auth Guard Is Wired But Does Not Handle JWT Expiry

**What goes wrong:**
The `proxy.ts` at `nemea-front/src/proxy.ts` IS correctly configured for Next.js 16 (named `proxy` export, correct matcher, `src/` location). The v1 senior review flagged "proxy.ts not wired as middleware.ts" -- this was actually a misdiagnosis. In Next.js 16, `middleware.ts` was renamed to `proxy.ts`, so the file IS the active proxy.

The real problem is different: `proxy.ts` checks `req.auth` (session exists) and `req.auth.accessToken` (backend token exists), but it CANNOT detect an expired JWT. The NextAuth session cookie may be valid while the backend JWT inside it has expired. When the user makes an API call:

1. `apiClientFetch` sends the expired JWT
2. Backend returns 401
3. `apiClientFetch` throws `new Error(error.message ?? 'API error: 401')`
4. Frontend shows a generic error toast
5. User has no idea they need to re-login

```typescript
// api-client.ts -- NO 401 handling
if (!res.ok) {
  const error = await res.json().catch(() => ({ message: `Error ${res.status}` }));
  throw new Error(error.message ?? `API error: ${res.status}`);
}
```

**Why it happens:**
NextAuth's JWT callback only fires during session creation (Google OAuth flow) or session refresh. If the backend JWT has a shorter TTL than the NextAuth session cookie, the proxy layer sees a valid session but the stored backend token is expired. The proxy cannot call the backend to verify -- it runs at the edge.

**Consequences:**
- Investor opens dashboard, works for an hour, JWT expires, every subsequent action shows "Error 401" toast with no recovery path
- User must manually navigate to /login or clear cookies -- terrible UX for a non-technical investor
- Admin editing Tiendanube config rates loses unsaved changes when API call fails silently

**Prevention:**
- Add 401 interception in `apiClientFetch`: on 401, call `signOut({ callbackUrl: '/login' })` to force re-authentication
- Better: implement a token refresh mechanism. On 401, try refreshing the NextAuth session (which re-triggers the `jwt` callback and exchanges a new Google id_token for a fresh backend JWT), then retry the original request once
- Set backend JWT TTL and NextAuth session maxAge to the same value, or make backend JWT longer-lived than the NextAuth session so the backend token never expires before the session does

**Detection:**
- Set backend JWT TTL to 60 seconds in dev. Use the app for 2 minutes. Every API call should fail silently after 60s.

**Phase to address:** Hardening phase (BEFORE Tiendanube/dashboard features). This is the #1 UX bug that will bite investor users who don't understand what "Error 401" means.

---

### Pitfall 3: Dashboard Aggregation Becomes a Performance Cliff

**What goes wrong:**
The investor dashboard needs to show a summary table: all products with cost, selling price, and margin. The current `ProductsController.findAll()` already calls BOTH `productsService.findAll()` AND `costsService.calculateAll()`:

```typescript
// products.controller.ts lines 60-63
const products = await this.productsService.findAll(includeInactive ?? false);
const costMap = await this.costsService.calculateAll();
```

Each of these fires 2 SQL queries (products + DISTINCT ON latest prices, BOM entries + DISTINCT ON latest supply prices). Total: 4 queries for the product list. This works fine for the admin product page.

But the dashboard needs MORE: it needs to add Tiendanube simulation data (commissions per plan/payment method) for each product. If implemented naively, the dashboard endpoint calls `costsService.calculateAll()` + a new `tiendanubeService.simulateAll(products, config)` that loops over products and runs the forward formula. For 50 products x 4 plans x 3 payment methods, that is 600 simulation runs in a single request.

**Why it happens:**
The "add a new service call" pattern is easy to follow. Each service works in isolation. But the dashboard is the first endpoint that needs to compose data from 3+ services.

**Consequences:**
- Dashboard response time: 200ms (acceptable) to 2s (unacceptable) depending on product count
- If Tiendanube simulation is done per-product in a loop, the endpoint becomes O(products x plans x payment_methods)
- Adding "filter by product type" or "sort by margin" requires re-running all calculations

**Prevention:**
- The Tiendanube forward calculation is PURE MATH (no DB calls). Pre-compute it in the service layer, not in the controller.
- Create a dedicated `DashboardService` that:
  1. Calls `costsService.calculateAll()` (2 queries)
  2. Calls `productsService.findAll()` (2 queries)
  3. Loads Tiendanube config ONCE (1 query)
  4. Runs forward simulation in-memory for the user's selected plan/payment config (a simple loop, no DB)
- Total: 5 queries regardless of product count, plus an O(n) in-memory loop
- Paginate if product count exceeds 100 (unlikely for a leather goods brand)

**Detection:**
- Log query count per request in NestJS `LoggingInterceptor`. If dashboard exceeds 10 queries, there's an N+1 hiding.

**Phase to address:** Dashboard/simulator phase. Design the `DashboardService` as a composition layer, not as another controller that chains service calls.

---

### Pitfall 4: Scenario Storage Leaking Into Real Product Data

**What goes wrong:**
Scenarios let investors override selling prices to simulate "what if I price this at $X instead of $Y". If scenarios are stored as actual price records in `product_price_history`, they pollute the real pricing data. A scenario price becomes the "latest price" for cost calculations, breaking the admin's view.

Similarly, if scenarios are stored by modifying the `products` table (even temporarily), concurrent access means one user's scenario affects another user's view.

**Why it happens:**
The simplest implementation is "change the price, see the result, change it back." But with append-only price history, "changing it back" means adding ANOTHER record, which compounds the pollution.

**Consequences:**
- Admin sees wrong product prices because an investor's scenario created a price history record
- Cost calculations use the scenario price as the "real" latest price
- Scenario cleanup is unreliable -- if the user closes the tab before "reverting," the dirty data persists

**Prevention:**
- Scenarios MUST be stored in a separate table: `scenarios` (id, user_id, name, created_at) + `scenario_overrides` (scenario_id, product_id, override_price)
- Scenario calculation is done entirely in memory: load real prices, then overlay the overrides, then run the Tiendanube formula
- NEVER write scenario data to `product_price_history` or modify the `products` table
- Scope scenarios to a user: `scenario.userId` foreign key. An investor can only see their own scenarios. Admin can see all.
- Database schema:

```sql
CREATE TABLE scenarios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  name VARCHAR(100) NOT NULL,
  config JSONB NOT NULL DEFAULT '{}',  -- plan, payment method, installments
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE scenario_overrides (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scenario_id UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  override_price DECIMAL(12,2) NOT NULL,
  UNIQUE(scenario_id, product_id)
);
```

**Detection:**
- Query `product_price_history` for records that don't correspond to admin actions. If any exist, scenarios are leaking.

**Phase to address:** Scenario/simulator phase. Design the schema BEFORE implementing the UI. The data isolation must be enforced at the DB level with foreign keys and separate tables, not with application-level conventions.

---

## Moderate Pitfalls

### Pitfall 5: DRY Cleanup Breaking Component Contracts

**What goes wrong:**
The v1 senior review identified:
- `SupplyOption` interface defined 8 times across 8 files
- `UNIT_LABELS` defined 5 times (supplies/types.ts, BomEditorDialog, BomGroupEditorDialog, ProductDetailClient, ProductExpandedRow)
- `formatDate` defined 4 times (products/PriceHistoryDialog, ProductExpandedRow, supplies/PriceHistoryDialog, SupplyExpandedRow)
- `PriceRecord` interface defined 2 times (supplies/types.ts, products/types.ts)

The temptation is to do a single "DRY cleanup PR" that extracts everything into shared modules. This breaks if:
1. The extracted types have subtle differences (e.g., `SupplyOption` in `page.tsx` has different fields than in `BomEditorDialog.tsx`)
2. The shared module creates import cycles
3. Components that were self-contained now depend on a shared barrel that bundles unrelated code

**Prevention:**
- Before deduplication, DIFF the 8 `SupplyOption` definitions. Current analysis shows they ARE identical: `{ id, name, unitType, type: { name }, isActive }`. Safe to extract.
- `UNIT_LABELS` in supplies/types.ts is typed as `Record<Supply['unitType'], string>` (strict union), while the product versions use `Record<string, string>` (loose). The extracted version should use the strict type.
- Create domain-specific shared files, not a single `types.ts` barrel:
  - `@/types/supply.ts` -- SupplyOption, UNIT_LABELS, formatPrice
  - `@/types/product.ts` -- Product, CostBreakdownItem, formatCost, formatMargin
  - `@/lib/formatters.ts` -- formatDate, formatAmount, formatMonthLabel
- Do NOT extract into `@/types/index.ts` -- barrel exports prevent tree-shaking and create circular dependency risk

**Detection:**
- After DRY cleanup, run `npm run build`. Any broken imports or type errors will surface immediately.
- Check bundle size before/after: the shared module should not increase the initial page bundle.

**Phase to address:** Hardening phase, BEFORE adding new features. New features will add more types, and cleaning up after is harder than cleaning up before.

---

### Pitfall 6: Binary Search Inverse Calculator Edge Cases

**What goes wrong:**
The `calcInverse` function in the prototype uses binary search between `costoProducto` and `costoProducto * 20` to find the selling price that yields a target profit:

```javascript
let low = costoProducto;
let high = costoProducto * 20;
for (let i = 0; i < 100; i++) { ... }
```

Edge cases that break this:
1. **Target profit higher than 20x markup allows:** If `gananciaDeseada` requires a price above `costoProducto * 20`, the binary search converges to the upper bound and returns a wrong answer silently.
2. **Negative target profit:** If investor enters a negative desired profit (selling at a loss), `low = costoProducto` starts above the answer. Binary search diverges.
3. **Zero cost product:** If `costoProducto = 0` (external production product with no BOM yet), `high = 0 * 20 = 0`, and the search space is empty.
4. **Transfer payment with no installments:** The forward formula has different code paths for `tarjeta` vs `transferencia`. If the binary search assumes one payment method but the forward formula switches conditionally, convergence is unpredictable.

**Prevention:**
- Use epsilon convergence, not fixed iterations: `while (high - low > 0.01)` with a max iteration guard (50 is more than enough for 0.01 precision on a $0-$10M range)
- Validate bounds: if `calcForward({ precioVenta: high }).gananciaReal < gananciaDeseada`, the target is unreachable. Return an error, don't return a wrong answer.
- Handle zero/negative cost: set `low = 0` and `high = max(gananciaDeseada * 5, 100000)` as a safe default
- The inverse function MUST use the exact same forward formula as the display. Extract `calcForward` as a shared pure function, never duplicate the formula.

**Detection:**
- Test: `calcInverse({ gananciaDeseada: 1000000, costoProducto: 100, ... })` should return an error or a valid price above $100,000, not $2,000 (the upper bound).

**Phase to address:** Calculator implementation phase. Write the edge case tests BEFORE implementing the inverse calculator.

---

### Pitfall 7: Tiendanube Rate Config Desynchronizing With Reality

**What goes wrong:**
Tiendanube/Pago Nube changes their commission rates periodically. As of March 2026, they introduced the SIRTAC regime (retention varying by seller's COMARB status and buyer's province). The current prototype hardcodes:

```javascript
const PLANS = {
  inicial: { tarjeta: { "1": 7.69, "7": 5.59, "14": 4.69 }, transferencia: 1.5, txTiendanube: 2.0 },
  // ...
};
const DEFAULT_CUOTAS_TASAS = { 1: 0, 3: 8.42, 6: 17.41, 9: 27.04, 12: 37.39 };
const IVA_RATE = 0.21;
```

If these are migrated as seed data into a `tiendanube_config` table, they will become stale the moment Tiendanube updates their rates. The user must update them manually, and there's no mechanism to alert them that rates might have changed.

**Prevention:**
- Make ALL rates admin-editable via a config UI, not just seeded once
- Add a `lastVerifiedAt` timestamp to the config. Show a warning in the calculator: "Tasas verificadas por ultima vez: DD/MM/YYYY" with a link to the official Pago Nube rates page
- Do NOT attempt to auto-fetch rates from Tiendanube (no public API for commission rates) -- this is explicitly out of scope
- IIBB/SIRTAC aliquot is per-seller, not per-plan. Store it as a single app-level config value (the user has one Tiendanube store). Don't try to model per-province SIRTAC rates -- the user's accountant handles the exact fiscal calculation. The app's goal is "good enough estimation."
- The config schema should support versioning: when rates change, keep the old rates for historical scenario comparison

**Detection:**
- Verify: if admin changes the "tarjeta 1 dia" rate from 7.69 to 8.00, does the calculator immediately reflect the new rate? If it requires a redeploy, the config is hardcoded somewhere.

**Phase to address:** Tiendanube config phase (admin-editable rates). Build the config BEFORE the calculator, so the calculator reads from DB from day one.

---

### Pitfall 8: Role-Based Frontend Routing Without Backend Enforcement

**What goes wrong:**
The sidebar currently shows ALL navigation items to ALL users (admin and investor alike). For v1.1, investors should only see Dashboard and Simulator. The naive implementation is:

```tsx
// BAD: only hide the sidebar link
{isAdmin && <Link href="/productos">Productos</Link>}
```

But if the investor navigates directly to `/productos`, the page renders because:
1. `proxy.ts` only checks auth/token presence, not roles
2. The `(app)/layout.tsx` renders children unconditionally
3. Backend endpoints like `GET /products` don't have `@Roles(Role.ADMIN)` -- they're accessible to any authenticated user

The investor sees all product data, costs, supplier names, and BOM details that should be admin-only.

**Prevention:**
- **Backend first:** Add `@Roles(Role.ADMIN)` to ALL existing controllers (products, supplies, suppliers, catalogs, expenses). Only the new dashboard/simulator endpoints should be accessible to USER role.
- **Frontend proxy:** Add role check in `proxy.ts` -- if `req.auth.user.role !== 'admin'` and path is not `/dashboard` or `/simulador`, redirect to `/dashboard`
- **Sidebar:** Conditional rendering is a UX convenience, not a security measure. Still do it, but don't rely on it.
- **API client:** `apiClientFetch` should handle 403 (Forbidden) differently from 401 (Unauthorized). 403 means "you don't have permission" (show message), 401 means "session expired" (redirect to login).

**Detection:**
- Test: log in as a USER role account. Navigate to `/productos` directly. If the page renders with data, role enforcement is missing.

**Phase to address:** Hardening phase. Lock down backend endpoints BEFORE adding new user-facing features. This is a security requirement, not a cosmetic one.

---

## Minor Pitfalls

### Pitfall 9: `deactivateBySupplier` Dead Code Creating False Confidence

**What goes wrong:**
`SuppliesService.deactivateBySupplier()` (line 275) exists in the backend but is never called by any controller or other service. It was likely planned for cascade deactivation when a supplier is deactivated. Its existence suggests the feature works, but it's unreachable code.

**Prevention:**
- Either wire it into `SuppliersService.toggleStatus()` (if the business rule is "deactivating a supplier deactivates all their supplies") or delete it
- Add a "dead code" lint check or simply `grep` for unused service methods as part of the DRY cleanup phase

**Phase to address:** Hardening phase (DRY cleanup).

---

### Pitfall 10: TypeORM decimal Columns Returned as Strings

**What goes wrong:**
TypeORM returns PostgreSQL `decimal(12,2)` columns as strings, not numbers. This is documented in [TypeORM issue #2937](https://github.com/typeorm/typeorm/issues/2937). The existing codebase already handles this in some places (`parseFloat(entry.quantity)` in `CostsService`, `parseFloat(row.price)` in cost calculations) but not consistently.

The frontend `Expense.amount` is typed as `string` (correct), and `formatAmount()` does `parseFloat(amount)`. But `PriceRecord.price` is also `string` and some components may display it raw without parsing.

When adding Tiendanube calculation service, if the service accepts a `productCost: number` parameter but receives a string from the cost service, the arithmetic silently coerces `"6534.48" * 1.21` which happens to work in JS but is type-unsafe and will fail TypeScript strict checks.

**Prevention:**
- Establish a convention: all monetary values in service interfaces are `number`. The conversion from string (TypeORM) to number happens at the repository/service boundary, not in the controller or frontend.
- Alternatively, use TypeORM column transformers to auto-convert:
  ```typescript
  @Column({ type: 'decimal', precision: 12, scale: 2, transformer: { to: (v: number) => v, from: (v: string) => parseFloat(v) } })
  ```
- For the Tiendanube service, use a dedicated DTO with explicit `number` types. Never pass raw entity fields into calculation functions.

**Phase to address:** Can be addressed incrementally during each feature phase. Establish the convention during Tiendanube config phase.

---

### Pitfall 11: Scenario Comparison UI Complexity Creep

**What goes wrong:**
The user wants to compare scenarios side-by-side. If the UI allows arbitrary scenario configurations (different plans, different payment methods, different price overrides), the comparison matrix becomes NxM where N is scenarios and M is products. The UI becomes a spreadsheet clone.

**Prevention:**
- V1.1 scope: ONE active scenario at a time, compared against "current real prices." Not N scenarios compared against each other.
- The scenario has a single Tiendanube config (plan + payment method + installments) and zero or more product price overrides
- Comparison view: two columns -- "Real" and "Scenario [name]". Not a pivot table.
- Defer multi-scenario comparison to v2

**Phase to address:** Simulator UI phase. Agree on the comparison UI wireframe BEFORE building the backend schema.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Hardening: middleware fix | Misdiagnosing proxy.ts as broken (it's NOT -- it's correct for Next.js 16) | Verify: `proxy.ts` exports named `proxy`, has `config.matcher`, is at `src/proxy.ts`. The REAL issue is 401 handling in `apiClientFetch`. |
| Hardening: 401 handling | Only handling 401 in `apiClientFetch`, forgetting `apiFetch` (server-side) | Both `api-client.ts` and `api.ts` need 401 handling, but with different strategies (client-side: redirect to login; server-side: throw to error boundary) |
| Hardening: DRY cleanup | Extracting types into shared module that breaks tree-shaking | Use domain-specific shared files (`@/types/supply.ts`, `@/types/product.ts`), not barrel exports |
| Hardening: roles | Adding @Roles to new endpoints but forgetting existing ones | Audit ALL existing controllers -- products, supplies, suppliers, catalogs, expenses all need `@Roles(Role.ADMIN)` added |
| Tiendanube config | Hardcoding rates as constants instead of reading from DB | Build config CRUD first, calculator reads from config service |
| Tiendanube config | Modeling SIRTAC/IIBB as a complex per-province table | Keep it simple: one aliquot percentage, admin-editable. The app is "good enough estimation." |
| Calculator: forward | Intermediate rounding losing centavos across 7 steps | Round only at the end of the full chain. Use centavo integers or Decimal.js. |
| Calculator: inverse | Binary search returning wrong answer silently on edge cases | Epsilon convergence + bounds validation + error return when target is unreachable |
| Dashboard | Calling 3+ services in sequence, each with 2+ queries | Create a DashboardService that composes data from a minimal number of queries + in-memory calculation |
| Scenario storage | Writing overrides to product_price_history | Separate `scenarios` + `scenario_overrides` tables. Never touch real pricing data. |
| Scenario UI | Allowing unlimited scenario complexity | V1.1: one active scenario vs reality. Defer multi-scenario comparison. |
| Role-based views | Frontend-only role enforcement | Backend `@Roles` on all endpoints + proxy.ts role-based redirects + conditional sidebar |

---

## "Looks Done But Isn't" Checklist: v1.1

- [ ] **401 redirect:** Expire a JWT while using the app. Does the user see "Error 401" toast or get redirected to login?
- [ ] **403 handling:** Log in as USER role. Navigate to `/productos`. Do you see data or get redirected?
- [ ] **Floating point:** Calculate forward at $87,000, then inverse targeting the resulting ganancia. Does the price round-trip within $0.01?
- [ ] **Binary search bounds:** Request inverse calc with ganancia = $10,000,000. Does it return an error or a wrong price?
- [ ] **Scenario isolation:** Create a scenario with override price. Check `product_price_history`. Are there new records? (There shouldn't be.)
- [ ] **Config editability:** Change a Tiendanube rate in the admin UI. Does the calculator use the new rate immediately without redeploy?
- [ ] **DRY types:** After cleanup, run `npm run build` on frontend. Zero type errors?
- [ ] **UNIT_LABELS consistency:** The shared UNIT_LABELS uses the strict union type, not `Record<string, string>`?
- [ ] **deactivateBySupplier:** Is it wired to a controller or deleted? No more dead code.
- [ ] **Dashboard perf:** Log SQL query count for dashboard endpoint. Is it <= 6?
- [ ] **proxy.ts role check:** Does `proxy.ts` redirect non-admin users away from admin-only routes?
- [ ] **Decimal handling:** Does the Tiendanube service receive numbers, not strings, from the cost service?

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Floating point errors in calculator | LOW | Fix formula to use integer centavos. No data corruption -- calculations are stateless. |
| Scenario data leaked to price history | HIGH | Manual DB cleanup: identify and delete scenario-originated price records. Must check which products were affected and whether the "latest price" is now correct. |
| Missing role enforcement on existing endpoints | MEDIUM | Add `@Roles(Role.ADMIN)` to all existing controllers. Deploy. Investor sessions immediately lose access to admin endpoints. May need to communicate to investor users that URLs they bookmarked no longer work. |
| Binary search returning wrong price | LOW | Fix algorithm, deploy. No persistent state. But if investor already made business decisions based on wrong price, that's a trust problem. |
| Tiendanube rates stale in DB | LOW | Admin updates rates via config UI. Historical scenarios show rates at time of creation (if versioning is implemented). |
| 401 not handled, users stuck | LOW | Deploy `apiClientFetch` fix. Immediate effect for all users. |

---

## Integration Points: Where Things Connect and Break

| Integration | Risk | What to Watch |
|-------------|------|---------------|
| CostsService -> TiendanubeService | Decimal string vs number mismatch | `CostsService.calculateAll()` returns `cost: number` but computed from `parseFloat(string)`. Pass as-is to TiendanubeService. |
| TiendanubeService -> DashboardService | Config load per product vs once | Load Tiendanube config ONCE in DashboardService, pass config object to Tiendanube calculations in a loop. Do NOT load config inside the loop. |
| ScenarioService -> ProductsService | Scenario must not call `productsService.addPrice()` | Scenarios use read-only access to products. Price overrides are in `scenario_overrides` table only. |
| proxy.ts -> apiClientFetch | 401 interception must not create infinite loops | If `signOut()` triggers a session check that triggers another 401, the page will loop. Guard with a `isRedirecting` flag. |
| Sidebar -> useIsAdmin hook | Session race condition on fresh login | `useIsAdmin()` depends on `useSession()` which is async. On fresh login, the first render may see `role: undefined`. Use loading state. |
| Backend RolesGuard -> new endpoints | Default behavior if no @Roles decorator | Currently, endpoints without `@Roles` are accessible to ALL authenticated users. New dashboard endpoints MUST specify `@Roles(Role.USER, Role.ADMIN)` explicitly, not rely on the default. |

---

## Sources

- [How to Handle Monetary Values in JavaScript -- Frontstuff](https://frontstuff.io/how-to-handle-monetary-values-in-javascript) -- floating point issues
- [Currency Calculations in JavaScript -- Honeybadger](https://www.honeybadger.io/blog/currency-money-calculations-in-javascript/) -- integer centavos pattern
- [TypeORM decimal column returns string -- Issue #2937](https://github.com/typeorm/typeorm/issues/2937) -- confirmed TypeORM behavior
- [How to properly handle decimals with TypeORM -- Medium](https://medium.com/@matthew.bajorek/how-to-properly-handle-decimals-with-typeorm-f0eb2b79ca9c) -- column transformers
- [Next.js 16 proxy.ts official docs](https://nextjs.org/docs/app/api-reference/file-conventions/middleware) -- confirms middleware renamed to proxy
- [NextAuth proxy.ts migration -- Discussion #13315](https://github.com/nextauthjs/next-auth/discussions/13315) -- `export { auth as proxy }` pattern
- [CVE-2025-29927: Next.js Middleware Authorization Bypass](https://projectdiscovery.io/blog/nextjs-middleware-authorization-bypass) -- proxy security considerations
- [Tiendanube comisiones y retenciones -- Centro de Ayuda](https://ayuda.tiendanube.com/es_ES/pago-nube/que-costos-debo-tener-en-cuenta-al-vender-con-pago-nube) -- SIRTAC regime, rate changes
- [Tiendanube planes y precios](https://www.tiendanube.com/planes-y-precios) -- current plan structure
- [Understanding the N+1 Problem in TypeORM -- Medium](https://medium.com/@bloodturtle/understanding-the-n-1-problem-in-typeorm-b93757dca974) -- dashboard query optimization
- [currency.js](https://currency.js.org/) -- lightweight monetary value library

---

*Pitfalls research for: v1.1 Tiendanube & Investor Dashboard on Nemea (NestJS + Next.js 16 + TypeORM + PostgreSQL)*
*Researched: 2026-03-26*
*Supersedes: v1 PITFALLS.md (2026-02-28) -- v1 pitfalls remain valid; this document covers v1.1-specific pitfalls only*
