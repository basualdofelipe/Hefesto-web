# Pitfalls Research

**Domain:** Product cost management / inventory system (NestJS + TypeORM + PostgreSQL + NextAuth)
**Researched:** 2026-02-28
**Confidence:** MEDIUM-HIGH (critical claims verified via official docs and multiple sources; some Railway-specific findings are MEDIUM based on community sources)

---

## Critical Pitfalls

### Pitfall 1: TypeORM `synchronize: true` leaking into production

**What goes wrong:**
`synchronize: true` in the TypeORM data source config auto-applies schema changes on app startup. In production this means a deploy can silently drop columns, alter types, or delete tables without a migration file ever being reviewed. Data is lost with no warning and no rollback path.

**Why it happens:**
Developers set `synchronize: true` for convenience in development (no migration needed), then either forget to disable it for production or accidentally deploy with the wrong environment variable. The flag reads from `process.env.NODE_ENV`, but if that env var is misconfigured on the CI/CD pipeline, the condition evaluates wrong.

**How to avoid:**
- Hardcode `synchronize: false` unconditionally. Do NOT use `synchronize: NODE_ENV !== 'production'`. Just always false.
- Use `migrationsRun: true` so migrations run automatically on startup (this is the safe equivalent).
- Add an assertion in `AppModule` startup: if `dataSource.options.synchronize === true` throw an error, so a misconfiguration fails loudly before touching the DB.
- Keep two separate DataSource configs: one for the NestJS app (runtime) and one for the TypeORM CLI (migration generation). They must both have `synchronize: false`.

**Warning signs:**
- `typeorm` CLI commands work but generate empty migration files — this often means synchronize is still doing the work silently.
- No migration files exist for schema changes that definitely happened.
- `npm run build && npx typeorm migration:generate` produces empty diffs.

**Phase to address:** Phase 2 — Backend scaffold. Establish the migration workflow before writing a single entity. Verified: multiple TypeORM docs and community posts confirm `synchronize: true` as the #1 cause of production data loss. [MEDIUM confidence — no official TypeORM incident report, but pattern is universally documented]

---

### Pitfall 2: Two separate TypeORM DataSource configs falling out of sync

**What goes wrong:**
NestJS loads TypeORM via `TypeOrmModule.forRoot()` with `autoLoadEntities: true`. The TypeORM CLI needs a separate `DataSource` export (typically in `src/data-source.ts`) that manually lists entities. When developers add a new entity and register it with `@Module` but forget to add it to the CLI config, `migration:generate` sees no changes because it has no knowledge of the new entity. The migration file is empty, the entity never lands in the database, and runtime errors appear only when a repository query fails.

**Why it happens:**
`autoLoadEntities` is a NestJS-specific feature that the raw TypeORM CLI cannot use. This creates a divergence: two configs that must stay in sync manually.

**How to avoid:**
- Keep the CLI `DataSource` in `src/data-source.ts` as the single source of truth for entities.
- In `TypeOrmModule.forRoot()`, do NOT use `autoLoadEntities: true`. Instead import the same entities array from `data-source.ts`. This guarantees one list.
- Add a test or lint rule that counts entity files vs entries in the DataSource config.

**Warning signs:**
- `npx typeorm migration:generate` consistently produces empty files even after adding a new entity and its `@Column` decorators.
- Runtime error: `EntityMetadataNotFoundError` for a recently added entity.

**Phase to address:** Phase 2 — Backend scaffold. Set up both configs correctly before implementing any entity.

---

### Pitfall 3: Google OAuth `id_token` ↔ NestJS JWT exchange done incorrectly

**What goes wrong:**
NextAuth handles Google OAuth and produces its own session JWT (a NextAuth-managed cookie). When the frontend calls the NestJS backend, it needs to send a token the backend can verify. There are two wrong approaches:
1. Passing the raw NextAuth session cookie to NestJS — the backend cannot verify it without NextAuth's secret and schema.
2. Passing Google's `access_token` to NestJS — Google's access tokens are opaque and not verifiable by your backend without a Google API call on every request.

The correct flow is: NextAuth exchanges the Google `id_token` (a signed JWT from Google) with your NestJS backend on first login. NestJS verifies the `id_token` with Google's public keys, creates a user record (or validates against the email whitelist), and mints its own JWT. That NestJS JWT is stored in the NextAuth session via the `jwt` callback and sent as `Authorization: Bearer` on every API call.

If this exchange is done wrong, you end up either with no auth on the backend, or an auth system that makes a Google API call on every request (N+1 latency per request to Google).

**Why it happens:**
There are three different tokens in play (Google's `id_token`, Google's `access_token`, NextAuth's session JWT) and documentation for each exists in isolation. Developers often conflate them.

**How to avoid:**
- In NextAuth `jwt` callback: call your NestJS `/auth/google` endpoint with `account.id_token`. Store the returned NestJS token as `token.backendToken`.
- In NextAuth `session` callback: expose `token.backendToken` as `session.accessToken`.
- NestJS `AuthModule`: implement a `POST /auth/google` endpoint that uses `google-auth-library` to verify the `id_token`, checks email against the whitelist table, and returns a signed JWT.
- NestJS guards then only verify the NestJS-minted JWT — never call Google on each request.

**Warning signs:**
- Frontend receives 401s from the backend even though NextAuth shows the user as logged in.
- Backend auth works, but requests take 200-400ms longer than expected (sign of Google verification happening per request).
- Session object on the frontend has no `accessToken` field.

**Phase to address:** Phase 3 — Auth. This is the highest-complexity phase. [MEDIUM confidence — based on multiple community sources and GitHub discussions, not official NextAuth docs]

---

### Pitfall 4: Email whitelist checked only at login, not on every request

**What goes wrong:**
The whitelist pattern (only approved emails can access the app) is implemented as a check during Google OAuth login. But if an email is later removed from the whitelist (e.g., someone leaves), their existing JWT is still valid until it expires. They can continue to make API calls for the JWT lifetime (days/weeks).

**Why it happens:**
OAuth-first thinking: "the login step controls access." In practice, the JWT is a bearer credential valid independently of the whitelist state.

**How to avoid:**
- NestJS JWT guard should check the database whitelist on every request — not just at login. With 2-3 users this is zero performance cost (one indexed lookup).
- Alternatively: keep JWT TTL short (1 hour) and implement refresh token rotation. Whitelist is re-checked on refresh.
- For this app's scale, checking the DB on every request is the simpler and safer choice.

**Warning signs:**
- Guard only validates JWT signature and expiry, never queries the DB.
- A deactivated user can still call authenticated endpoints.

**Phase to address:** Phase 3 — Auth.

---

### Pitfall 5: "Latest price per supply" query without the right index (N+1 or full table scan)

**What goes wrong:**
The core cost calculation is: `cost = SUM(latest_price(supply_id) * quantity)`. If implemented naively, this becomes an N+1 query — one query per supply in the product's composition. For a product with 8 supplies, that's 8 subqueries. For a page listing 20 products, that's 160 queries. PostgreSQL will execute full table scans on `supplies_price_history` for each one without the right index.

The naive TypeORM approach is:
```typescript
const latestPrice = await this.priceHistoryRepo.findOne({
  where: { supply: { id: supplyId } },
  order: { created_at: 'DESC' }
});
```
This fires one query per supply.

**Why it happens:**
The ORM makes it easy to query individual records. The cost-of-multiple-queries problem only becomes visible when you look at the NestJS logger or a query profiler.

**How to avoid:**
- Use a single SQL query with `DISTINCT ON (supply_id) ORDER BY supply_id, created_at DESC` to get the latest price for all supplies in one shot.
- Add a composite index on `supplies_price_history(supply_id, created_at DESC)` — this is mandatory, not optional.
- In TypeORM, implement this as a `createQueryBuilder` raw query or a PostgreSQL view, not via the repository API.
- For the product cost endpoint, load the entire product composition + latest prices in one JOIN, not in separate calls.

**Warning signs:**
- NestJS logger shows many similar `SELECT ... FROM supplies_price_history WHERE supply_id = $1` queries per request.
- Cost calculation endpoint takes >200ms on a product with 5+ supplies.

**Phase to address:** Phase 5 — Costos (cost calculation). Design the query correctly from the start, before it's used anywhere.

Note: PostgreSQL DISTINCT ON does a full index scan for deduplication — it cannot "skip" to the next group key. The correct index (`supply_id, created_at DESC`) still reduces this to an index-only scan, which is fast enough for Nemea's scale. TimescaleDB SkipScan (which truly optimizes this) is unnecessary at this scale. [MEDIUM confidence — based on PostgreSQL docs and community benchmarks]

---

### Pitfall 6: Soft delete with `is_active` breaking unique constraints

**What goes wrong:**
Products, supplies, and suppliers use soft delete (`is_active = false`). If a supply named "Cuero Rojo 3mm" is deactivated and then a new one with the same name is created, a `UNIQUE` constraint on `(name, supplier_id)` will reject the new insert because the old (deactivated) record still exists.

This makes soft delete semantics clash with uniqueness: you want "one active supply per name+supplier" but the DB enforces "one supply ever per name+supplier."

**Why it happens:**
TypeORM's `@Unique` decorator creates a standard unique index, which does not filter by `is_active`. This is correct SQL behavior — NULL is treated differently, but `false` is not ignored.

**How to avoid:**
Option A (recommended for PostgreSQL): Use TypeORM's `@DeleteDateColumn` (`deleted_at TIMESTAMPTZ`) instead of `is_active`. Then create a partial unique index: `CREATE UNIQUE INDEX idx_supplies_name_unique WHERE deleted_at IS NULL`. TypeORM supports this via `@Index(['name', 'supplier_id'], { where: '"deletedAt" IS NULL', unique: true })`.

Option B (if keeping `is_active`): Create a partial unique index in a migration: `CREATE UNIQUE INDEX idx_supplies_name_active WHERE is_active = true`. TypeORM cannot declare this declaratively — must be raw SQL in a migration.

The `is_active` approach (Option B) requires manually filtering every query (`where: { isActive: true }`), while `@DeleteDateColumn` auto-filters via TypeORM's soft-delete support.

**Warning signs:**
- `QueryFailedError: duplicate key value violates unique constraint` when trying to create a supply with the same name as a previously deactivated one.
- Unit tests that create → soft delete → create same entity start failing.

**Phase to address:** Phase 4 — ABM Insumos/Proveedores. Decide between `is_active` and `deleted_at` during entity design, before data exists. [HIGH confidence — TypeORM GitHub issue #7549 and wanago.io article confirm this is a known problem with a documented solution]

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| `synchronize: true` in dev | No migration step during active entity design | Dev and prod behavior diverge; devs forget to generate migrations | Never — use `migrationsRun: true` with real migrations from day 1 |
| Skipping the composite index on `supplies_price_history` | Simpler setup | Cost calculation is slow from day 1, worsens with data | Never — add the index in the migration that creates the table |
| Checking email whitelist only at login | Simpler guard code | Deactivated users retain access for JWT lifetime | Never for a multi-user internal app |
| On-the-fly cost calculation (no cache) | Always fresh, no invalidation logic | Fine for Nemea's scale; only a problem at thousands of products | Acceptable — defer caching decision to V2 if needed |
| `is_active` boolean for soft delete | Intuitive for devs | Breaks unique constraints; not handled automatically by TypeORM | Acceptable if partial unique indexes are created in the same migration |
| Exposing NestJS server errors in HTTP responses | Easier debugging | Leaks schema/stack info to clients | Never in production — use NestJS exception filters |
| JWT with long TTL (7+ days) | Fewer re-logins | Revocation is hard; deactivated users stay active | Never — use 1h access token + refresh token |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| NextAuth → NestJS | Pass NextAuth session cookie directly to NestJS | Use `jwt` callback to exchange Google `id_token` for NestJS JWT; store NestJS JWT in session |
| TypeORM CLI ↔ NestJS | Use `autoLoadEntities` for both | Two configs: NestJS uses `autoLoadEntities`, CLI uses explicit entities list imported from a shared `data-source.ts` |
| Docker Compose → NestJS local | Use `localhost` as DB host | Use Docker service name (`postgres`) as host when app runs inside Docker; use `127.0.0.1` only when app runs on host machine |
| Railway PostgreSQL | Missing SSL config | Add `ssl: { rejectUnauthorized: false }` in TypeORM config for Railway connections (Railway requires SSL) |
| `depends_on` in Docker Compose | Assume DB is ready when container starts | Add a health check or retry logic — `depends_on` only waits for container start, not PostgreSQL readiness |
| Google `id_token` → NestJS | Verify with `google-auth-library` on every request | Verify once on login, mint a NestJS JWT, verify the NestJS JWT on every subsequent request |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| N+1 on `latest_price(supply_id)` | Cost endpoint slow; many similar queries in NestJS logger | Single `DISTINCT ON` query with composite index | With >5 products loaded in one page |
| Missing index on `supplies_price_history(supply_id, created_at DESC)` | Full table scans on every cost calculation | Add index in the migration that creates the table | Even at 100 price records it's noticeable |
| No index on `supplies_per_product_history(product_id, is_active)` | Slow product composition load | Add composite index for the most common query pattern | At >200 product variants |
| Fetching full product list with cost calculation for each | Product list page takes seconds | Paginate; calculate cost only on product detail, not list | At >50 products |
| Unguarded `withDeleted` TypeORM option | Accidentally returning soft-deleted records | Default to never using `withDeleted`; only use explicitly in admin restoration flows | From day 1 — a logic bug, not a scale problem |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Storing NestJS JWT secret in `.env` but also in git history | Anyone with repo access can forge JWTs | Use Railway environment variables for secrets; never commit `.env` files with real secrets |
| Not validating Google `id_token` audience claim | Any valid Google token (from any app) accepted | Verify `aud` claim matches your Google OAuth client ID in `google-auth-library` verification |
| Email whitelist checked only at login | Deactivated users retain API access for JWT lifetime | Check whitelist on every request in the JWT guard |
| Role sent from frontend in request body | Attacker can self-elevate to ADMIN | Role always read from the JWT payload (set by backend at login), never from request body |
| Global validation pipe not configured | Malformed request bodies reach controllers | Apply `ValidationPipe` globally with `whitelist: true, forbidNonWhitelisted: true` from day 1 |
| Exposing raw TypeORM/PostgreSQL errors | Schema and query structure leaks to client | Implement a global `ExceptionFilter` that maps DB errors to generic HTTP responses |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Cost calculation shows stale price without indicating when it was last updated | User doesn't know if cost reflects today's prices | Show `precio actualizado: DD/MM/YYYY` next to each cost; include `supply.latest_price_date` in the response |
| Soft-deleted supplies still appear in product composition form | Confusing — user tries to add a deactivated material | Filter supply selector to `is_active = true` only in the frontend |
| No confirmation before deactivating a supply used in active products | User accidentally breaks cost calculation for multiple products | Before soft-deleting a supply, query which products use it and show a warning with product names |
| Price history shows all records unsorted | Hard to understand price evolution | Always sort price history by `created_at DESC`; show the latest price first |
| Saving a new price without seeing impact on product costs | User unaware that updating price X affects Y products | After price update, show: "Este precio impacta N productos: [lista]" |

---

## "Looks Done But Isn't" Checklist

- [ ] **TypeORM migrations:** Migration files exist AND `npm run typeorm migration:run` executes cleanly on a fresh DB — verify both
- [ ] **Auth guard:** Returns 401 (not 500) for missing/expired/invalid tokens — test with curl and bad/no token
- [ ] **Email whitelist:** Removing an email from DB revokes access on the next request, not on the next login — test by deleting the DB record and retrying an authenticated call
- [ ] **Soft delete:** Deactivated supplies do NOT appear in "add material to product" dropdowns — verify the frontend filter is applied
- [ ] **Unique constraints with soft delete:** Creating a supply with the same name as a deactivated one succeeds — run this specific test case
- [ ] **Cost calculation:** Shows the correct price when the same supply has 3+ price history records — verify it always uses the most recent
- [ ] **Docker Compose:** `docker compose down && docker compose up` produces a clean, working state (no data corruption from incomplete shutdown)
- [ ] **Railway SSL:** Production deploy connects to PostgreSQL without SSL errors — verify `ssl: { rejectUnauthorized: false }` in Railway config
- [ ] **Role enforcement:** A USER-role JWT cannot call ADMIN-only endpoints — verify with a test JWT that has `role: USER`
- [ ] **Migration auto-run:** After deploy to Railway, migrations run automatically without manual intervention — verify `migrationsRun: true` in production config

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| `synchronize: true` ran in production and dropped columns | HIGH | Restore from last Railway PostgreSQL backup; replay transactions from app logs if available; add `synchronize: false` before re-deploying |
| Wrong `DISTINCT ON` query returns stale prices | LOW | Fix query, deploy; no data corruption — price history data is append-only |
| Soft delete + unique constraint conflict breaks inserts | MEDIUM | Write a migration that drops the standard unique index and creates a partial index `WHERE is_active = true`; deploy without downtime |
| Email whitelist bypass (deactivated user retains access) | LOW | Change JWT secret to invalidate all existing tokens; re-deploy; all users must log in again |
| Circular dependency in NestJS modules | LOW | Use `forwardRef()` as temporary fix; refactor to extract shared service into a `SharedModule` as permanent fix |
| Railway SSL connection refused in production | LOW | Add `ssl: { rejectUnauthorized: false }` to TypeORM config; redeploy (Railway docs confirm this is required) |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| `synchronize: true` in production | Phase 2 — Backend scaffold | Run `npm run typeorm migration:generate` after adding first entity; confirm file is non-empty |
| Two TypeORM DataSource configs out of sync | Phase 2 — Backend scaffold | Both configs reference same entities array; CI generates migration and fails if it's empty when it shouldn't be |
| Google OAuth ↔ NestJS token exchange | Phase 3 — Auth | Postman test: exchange a real Google `id_token` for a NestJS JWT; guard returns 200 with valid token, 401 with invalid |
| Email whitelist checked only at login | Phase 3 — Auth | Integration test: create user, delete from whitelist, make authenticated API call — expect 401 |
| N+1 on latest price query | Phase 5 — Costos | Query log shows ONE query per cost calculation request, not one per supply; add this as an acceptance criterion |
| Missing composite index on price history | Phase 4 — Insumos/Precios | Index is created in the same migration as the table — code review checklist item |
| Soft delete breaking unique constraints | Phase 4 — ABM Insumos | Test: create supply → deactivate → create same supply again; expect success |
| Railway SSL missing in production | Phase 2 — Backend scaffold (prod config) | First Railway deploy connects to DB without errors |
| Role not enforced on every endpoint | Phase 3 — Auth | Role guard test: USER token cannot hit admin endpoints; expect 403 |
| Docker `depends_on` not waiting for DB | Phase 2 — Backend scaffold | `docker compose up` results in app successfully connecting to PostgreSQL on first attempt; add health check if needed |

---

## Sources

- TypeORM migrations official docs and community articles: migrations over synchronize pattern — [Medium: Migrations Over Synchronize in TypeORM](https://medium.com/swlh/migrations-over-synchronize-in-typeorm-2c66bc008e74)
- TypeORM two DataSource configs issue: [NestJS & TypeORM Migrations in 2025](https://javascript.plainenglish.io/nestjs-typeorm-migrations-in-2025-50214275ec8d)
- NextAuth Google OAuth + custom backend token exchange: [Next Auth google OAuth with custom backend access token · Discussion #8884](https://github.com/nextauthjs/next-auth/discussions/8884)
- NextAuth JWT callback pattern: [Using Google OAuth 2.0 with a custom backend in Next.js](https://dev.to/udassi/using-google-oauth-20-with-a-custom-backend-in-nextjs-596d)
- Soft delete + unique constraint in TypeORM: [Unique Index and soft delete · Issue #7549](https://github.com/typeorm/typeorm/issues/7549), [wanago.io soft deletes with PostgreSQL and TypeORM](https://wanago.io/2021/10/25/api-nestjs-soft-deletes-postgresql-typeorm/)
- PostgreSQL DISTINCT ON performance for latest-per-group: [The Fastest Way to Get the Most Recent Row Per Group in Postgres](https://ellisvalentiner.com/post/2023-01-07-the-fastest-way-to-get-the-most-recent-row-per-group-in-postgres/), [Tiger Data: Select the Most Recent Record](https://www.tigerdata.com/blog/select-the-most-recent-record-of-many-items-with-postgresql)
- Docker Compose PostgreSQL readiness and host config: [NestJS + Docker + PostgreSQL Debugging](https://dev.to/diana_tang/nestjs-docker-postgresql-debugging-why-is-it-trying-to-connect-as-callustang-1g5m)
- Railway SSL for PostgreSQL: [Railway Help Station: SSL issues connecting to Railway Postgres](https://station.railway.com/questions/unable-to-connect-to-railway-postgres-fr-03e6edff)
- NestJS circular dependency: [NestJS official docs: circular dependency](https://docs.nestjs.com/fundamentals/circular-dependency), [Trilon: Avoiding Circular Dependencies in NestJS](https://trilon.io/blog/avoiding-circular-dependencies-in-nestjs)
- NestJS security: JWT guard and validation pipe — [NestJS official security docs](https://docs.nestjs.com/security/authentication)

---
*Pitfalls research for: NestJS + TypeORM + PostgreSQL + NextAuth cost management system*
*Researched: 2026-02-28*
