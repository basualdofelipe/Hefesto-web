# Phase 2: Auth - Context

**Gathered:** 2026-03-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Users can log in with Google and the backend enforces authentication and roles on every request. Covers: Google OAuth via NextAuth, token exchange with NestJS, global JWT guard, email whitelist in DB, and ADMIN/USER roles. Frontend auth UI (login page, session handling, logout) is included. User management UI is NOT included (endpoints only).

</domain>

<decisions>
## Implementation Decisions

### Login flow and session
- Dedicated `/login` page with Nemea branding and "Ingresar con Google" button
- Redirect-based OAuth flow (not popup) — standard, reliable, mobile-friendly
- Session persists across browser restarts (NextAuth HTTP-only cookie)
- After login, redirect to the page the user originally tried to access (callbackUrl pattern)

### Whitelist and onboarding
- First ADMIN email seeded via a TypeORM migration — reproducible, runs automatically
- Scale: 2-3 users in V1 (1 ADMIN + 1-2 USERs)
- Non-whitelisted users see a dedicated "access denied" page after Google OAuth — "Tu cuenta no tiene acceso a Nemea. Contactá al administrador." with a logout button
- Whitelist management: backend endpoints only (POST/DELETE /users), no UI in this phase. ADMIN manages via Swagger in dev or API calls

### Roles and permissions
- Two roles only: ADMIN (full CRUD) and USER (read-only)
- UI for USER role: hide action buttons entirely (no disabled buttons) — clean read-only experience
- @Public() decorator for opt-out pattern: JwtAuthGuard is global (APP_GUARD), public routes explicitly marked
- Public routes: GET /health (Railway needs it), Swagger /api/docs (dev only, already gated by NODE_ENV)

### Token management and security
- Whitelist checked on every request in the JWT guard (not just at login) — instant revocation, zero perf cost at 2-3 users
- JWT TTL: 7 days (longer is fine because whitelist is checked every request anyway)
- No refresh token needed (whitelist-per-request makes short TTL unnecessary)
- JWT payload: minimal — `sub` (user_id), `email`, `role` (ADMIN/USER)
- Session expiry UX: silent redirect to /login when 401 received. Google remembers the session so re-login is one click
- Logout: dropdown menu from user avatar in header — shows name/avatar + "Cerrar sesión" option

### Claude's Discretion
- Login page visual design and layout
- Exact error messages and copy
- NextAuth session/JWT callback implementation details
- Guard implementation patterns (metadata key naming, etc.)
- How to handle edge cases (network errors during OAuth, etc.)

</decisions>

<specifics>
## Specific Ideas

- The login page should feel branded (Nemea logo, fonts) — not a generic auth page
- The "access denied" page should be friendly, not scary — the user did nothing wrong, they just aren't on the list
- User avatar in the header comes from Google profile data (picture URL)

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BaseEntity` (`nemea-back/src/common/entities/base.entity.ts`): Provides id, created_at, updated_at — User entity should extend this
- `HttpExceptionFilter` (`nemea-back/src/common/filters/`): Already catches all exceptions — auth errors will be properly formatted
- `ResponseInterceptor` (`nemea-back/src/common/interceptors/`): Wraps responses in envelope — auth responses will be consistent
- `LoggingInterceptor`: Request logging already in place — auth requests will be logged
- Shadcn/ui components (`nemea-front/src/components/ui/`): Button, Card, Input, Sonner — available for login page

### Established Patterns
- `APP_GUARD` pattern already used for ThrottlerGuard — add JwtAuthGuard and RolesGuard to the same providers array
- Env validation via Joi (`env.validation.ts`) — add JWT_SECRET, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET
- Shared DataSource (`data-source.ts`) with glob entity loading — new User entity auto-discovered
- `synchronize: false` + `migrationsRun: true` — migration workflow established
- Global prefix `api` — all auth endpoints will be `/api/auth/*`
- ValidationPipe with whitelist + forbidNonWhitelisted — DTOs auto-validated

### Integration Points
- `AppModule` providers array: add JwtAuthGuard as APP_GUARD (alongside existing ThrottlerGuard)
- `AppModule` imports: add AuthModule (new module with JwtModule, PassportModule)
- `main.ts` CORS config: already reads FRONTEND_URL from env — NextAuth callbacks will work
- Frontend `layout.tsx`: wrap with NextAuth SessionProvider
- Frontend `next.config.mjs`: NEXTAUTH_URL and NEXTAUTH_SECRET env vars already templated

</code_context>

<deferred>
## Deferred Ideas

- User management UI (list users, change roles, add/remove from whitelist) — future phase or V2
- Multiple auth providers (GitHub, email/password) — V2 if needed
- Audit log of login events — V2

</deferred>

---

*Phase: 02-auth*
*Context gathered: 2026-03-01*
