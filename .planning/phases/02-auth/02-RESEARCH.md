# Phase 2: Auth - Research

**Researched:** 2026-03-01
**Domain:** Authentication & Authorization (Google OAuth + JWT + RBAC)
**Confidence:** HIGH

## Summary

Phase 2 implements Google OAuth login via Auth.js v5 (NextAuth) on the Next.js 16 frontend, with a NestJS backend that mints its own JWT after verifying Google's `id_token`. The backend enforces authentication globally via `APP_GUARD` with a `@Public()` opt-out decorator, and authorization via a `RolesGuard` checking `@Roles()` metadata. The email whitelist is checked on every request (not just at login) for instant revocation.

The architecture involves three distinct tokens: Google's `id_token` (signed JWT from Google), NestJS's backend JWT (minted by our backend after whitelist verification), and NextAuth's session cookie (HTTP-only, stores the backend JWT internally). The frontend never sends the NextAuth cookie to the backend -- it extracts the backend JWT from the session and sends it as `Authorization: Bearer <token>`.

**Primary recommendation:** Use `next-auth@beta` (Auth.js v5) with the Google provider on the frontend, `@nestjs/jwt` + `@nestjs/passport` + `passport-jwt` + `google-auth-library` on the backend. The backend's `POST /api/auth/google` endpoint verifies Google's `id_token`, checks the email whitelist, and returns a signed JWT. The JWT guard + whitelist check runs on every request; the roles guard checks `@Roles()` metadata.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Dedicated `/login` page with Nemea branding and "Ingresar con Google" button
- Redirect-based OAuth flow (not popup) -- standard, reliable, mobile-friendly
- Session persists across browser restarts (NextAuth HTTP-only cookie)
- After login, redirect to the page the user originally tried to access (callbackUrl pattern)
- First ADMIN email seeded via a TypeORM migration -- reproducible, runs automatically
- Scale: 2-3 users in V1 (1 ADMIN + 1-2 USERs)
- Non-whitelisted users see a dedicated "access denied" page after Google OAuth -- "Tu cuenta no tiene acceso a Nemea. Contacta al administrador." with a logout button
- Whitelist management: backend endpoints only (POST/DELETE /users), no UI in this phase. ADMIN manages via Swagger in dev or API calls
- Two roles only: ADMIN (full CRUD) and USER (read-only)
- UI for USER role: hide action buttons entirely (no disabled buttons) -- clean read-only experience
- @Public() decorator for opt-out pattern: JwtAuthGuard is global (APP_GUARD), public routes explicitly marked
- Public routes: GET /health (Railway needs it), Swagger /api/docs (dev only, already gated by NODE_ENV)
- Whitelist checked on every request in the JWT guard (not just at login) -- instant revocation, zero perf cost at 2-3 users
- JWT TTL: 7 days (longer is fine because whitelist is checked every request anyway)
- No refresh token needed (whitelist-per-request makes short TTL unnecessary)
- JWT payload: minimal -- `sub` (user_id), `email`, `role` (ADMIN/USER)
- Session expiry UX: silent redirect to /login when 401 received. Google remembers the session so re-login is one click
- Logout: dropdown menu from user avatar in header -- shows name/avatar + "Cerrar sesion" option

### Claude's Discretion
- Login page visual design and layout
- Exact error messages and copy
- NextAuth session/JWT callback implementation details
- Guard implementation patterns (metadata key naming, etc.)
- How to handle edge cases (network errors during OAuth, etc.)

### Deferred Ideas (OUT OF SCOPE)
- User management UI (list users, change roles, add/remove from whitelist) -- future phase or V2
- Dynamic RBAC: table `role_permissions` with UI to assign granular permissions per role per resource -- V2+
- Multiple auth providers (GitHub, email/password) -- V2 if needed
- Audit log of login events -- V2
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| AUTH-01 | User can log in with Google OAuth and stay logged in across sessions | Auth.js v5 Google provider + JWT session strategy + HTTP-only cookie persistence. `proxy.ts` protects routes. Session survives browser restart via cookie. |
| AUTH-02 | Only whitelisted emails in DB can access the app (no public registration) | Backend `POST /api/auth/google` checks `users` table before minting JWT. NextAuth `signIn` callback returns false if backend rejects. Frontend shows "access denied" page. |
| AUTH-03 | User with ADMIN role can create, edit, and delete all resources | `RolesGuard` checks `@Roles(Role.ADMIN)` decorator on write endpoints. JWT payload includes `role` field. |
| AUTH-04 | User with USER role can view all resources but cannot modify anything | Default role is USER (read-only). Write endpoints require `@Roles(Role.ADMIN)`. Read endpoints have no role restriction (any authenticated user). Frontend conditionally hides action buttons based on session role. |
| AUTH-05 | Backend validates JWT from NextAuth on every request and checks email whitelist | `JwtAuthGuard` registered as `APP_GUARD` validates JWT signature + expiry, then queries `users` table for email whitelist + active status. Fails with 401 if not found. `@Public()` decorator opts out specific routes. |
</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `next-auth` | 5.x (beta) | Frontend auth - Google OAuth, session management, JWT callbacks | Official Auth.js for Next.js. App Router native, supports `proxy.ts` in Next.js 16. Only production-grade auth library for Next.js ecosystem. |
| `@nestjs/jwt` | ^11.0.2 | Backend JWT signing and verification | Official NestJS JWT module. Wraps `jsonwebtoken`. Integrates with DI. |
| `@nestjs/passport` | ^11.0.5 | Passport integration for NestJS guards | Official NestJS Passport wrapper. Provides `AuthGuard()` base class. |
| `passport-jwt` | ^4.0.1 | JWT extraction and validation strategy | Standard Passport strategy for Bearer token auth. Extracts JWT from `Authorization` header. |
| `google-auth-library` | ^9.x | Backend verification of Google `id_token` | Official Google library. Verifies signature against Google's public keys without calling Google API per request. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `passport` | ^0.7.x | Core Passport framework | Peer dependency of `@nestjs/passport` and `passport-jwt`. |
| `@types/passport-jwt` | ^4.0.x | TypeScript types for passport-jwt | Dev dependency for type safety. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `next-auth@beta` (Auth.js v5) | `next-auth@4` (stable) | v4 is stable but uses legacy `middleware.ts` (deprecated in Next.js 16), pages router patterns, and `getServerSession()` instead of `auth()`. v5 is App Router native and works with `proxy.ts`. |
| `google-auth-library` for id_token verification | Manual JWT verification with `jwks-rsa` | google-auth-library handles key rotation, caching, and clock skew automatically. Manual approach requires managing JWKS endpoint, key caching, and retry logic. |
| `@nestjs/passport` + `passport-jwt` | Manual JWT verification with `@nestjs/jwt` only | Passport provides structured strategy pattern and integrates with NestJS guard system. Manual approach works but loses the `AuthGuard` base class and established patterns. |

**Installation:**

Backend:
```bash
cd nemea-back
npm install @nestjs/jwt @nestjs/passport passport passport-jwt google-auth-library
npm install -D @types/passport-jwt
```

Frontend:
```bash
cd nemea-front
npm install next-auth@beta
```

## Architecture Patterns

### Recommended Project Structure

**Backend additions:**
```
nemea-back/src/
├── auth/
│   ├── auth.module.ts              # AuthModule - imports JwtModule, PassportModule, TypeOrmModule
│   ├── auth.controller.ts          # POST /auth/google (token exchange), GET /auth/me
│   ├── auth.service.ts             # Google id_token verification, JWT minting, whitelist check
│   ├── strategies/
│   │   └── jwt.strategy.ts         # PassportStrategy(Strategy, 'jwt') - extracts + validates JWT
│   ├── guards/
│   │   ├── jwt-auth.guard.ts       # Extends AuthGuard('jwt') + @Public() check + whitelist check
│   │   └── roles.guard.ts          # Checks @Roles() metadata against request.user.role
│   ├── decorators/
│   │   ├── public.decorator.ts     # @Public() - SetMetadata(IS_PUBLIC_KEY, true)
│   │   ├── roles.decorator.ts      # @Roles(...roles) - SetMetadata(ROLES_KEY, roles)
│   │   └── current-user.decorator.ts # @CurrentUser() - extracts user from request
│   └── dto/
│       ├── google-auth.dto.ts      # { idToken: string }
│       └── auth-response.dto.ts    # { accessToken: string, user: UserProfile }
├── users/
│   ├── users.module.ts             # UsersModule - TypeOrmModule.forFeature([User])
│   ├── users.service.ts            # findByEmail, create, delete, findAll
│   ├── users.controller.ts         # POST /users, DELETE /users/:id (ADMIN only)
│   ├── entities/
│   │   └── user.entity.ts          # User entity extending BaseEntity
│   └── dto/
│       └── create-user.dto.ts      # { email: string, role: Role, name?: string }
├── database/
│   ├── data-source.ts              # (existing)
│   └── migrations/
│       └── XXXXXX-CreateUserTable.ts  # User table + seed ADMIN email
```

**Frontend additions:**
```
nemea-front/src/
├── auth.ts                         # NextAuth config - Google provider, JWT callbacks
├── proxy.ts                        # Route protection (replaces middleware.ts in Next.js 16)
├── app/
│   ├── api/
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts        # NextAuth route handler: export { GET, POST } from handlers
│   ├── login/
│   │   └── page.tsx                # Branded login page with Google button
│   ├── acceso-denegado/
│   │   └── page.tsx                # Access denied page for non-whitelisted users
│   └── layout.tsx                  # (modify) Wrap with SessionProvider
├── providers/
│   ├── ThemeProvider.tsx            # (existing)
│   └── SessionProvider.tsx          # "use client" wrapper for NextAuth SessionProvider
├── lib/
│   ├── utils.ts                    # (existing)
│   └── api.ts                      # Fetch wrapper that attaches Authorization: Bearer header
└── types/
    └── next-auth.d.ts              # Module augmentation for Session, JWT, User types
```

### Pattern 1: Token Exchange Flow

**What:** Frontend completes Google OAuth via NextAuth, then exchanges the Google `id_token` with the NestJS backend for a backend-minted JWT.
**When to use:** On every new login (first time the `jwt` callback fires with an `account` object).

```typescript
// Source: Auth.js official docs + nextauthjs/next-auth#8884
// auth.ts (frontend)
import NextAuth from 'next-auth';
import Google from 'next-auth/providers/google';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [Google],
  session: { strategy: 'jwt' },
  pages: {
    signIn: '/login',
    error: '/login',
  },
  callbacks: {
    async signIn({ account }): Promise<boolean | string> {
      if (account?.provider === 'google') {
        try {
          const res = await fetch(
            `${process.env.NEXT_PUBLIC_API_URL}/api/auth/google`,
            {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ idToken: account.id_token }),
            },
          );
          if (!res.ok) return '/acceso-denegado';
          return true;
        } catch {
          return false;
        }
      }
      return false;
    },
    async jwt({ token, account }): Promise<typeof token> {
      if (account?.provider === 'google') {
        // Exchange Google id_token for backend JWT
        const res = await fetch(
          `${process.env.NEXT_PUBLIC_API_URL}/api/auth/google`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ idToken: account.id_token }),
          },
        );
        if (res.ok) {
          const data = await res.json();
          token.backendToken = data.data.accessToken;
          token.role = data.data.user.role;
          token.userId = data.data.user.id;
        }
      }
      return token;
    },
    async session({ session, token }): Promise<typeof session> {
      session.accessToken = token.backendToken as string;
      session.user.role = token.role as string;
      session.user.id = token.userId as string;
      return session;
    },
  },
});
```

**Important note:** The `signIn` callback and `jwt` callback both fire during the sign-in flow. The `signIn` callback runs first and determines if sign-in should proceed. The `jwt` callback runs after and can store the backend token. To avoid calling the backend twice, the planner should consider either: (a) letting signIn just return true/false and doing the full exchange in jwt, or (b) storing the result in a closure. The simplest approach is to call the backend in the `jwt` callback only (when `account` is present) and handle the rejection by checking the response.

### Pattern 2: Global JWT Guard with @Public() Opt-Out

**What:** All routes require authentication by default. Routes marked with `@Public()` skip the guard.
**When to use:** This is the global security architecture.

```typescript
// Source: NestJS official docs + verified community patterns
// public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = (): ReturnType<typeof SetMetadata> =>
  SetMetadata(IS_PUBLIC_KEY, true);

// jwt-auth.guard.ts
import { ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private readonly reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context) as Promise<boolean>;
  }

  handleRequest<TUser>(err: Error | null, user: TUser | false): TUser {
    if (err || !user) {
      throw err || new UnauthorizedException('Token invalido o expirado');
    }
    return user;
  }
}
```

### Pattern 3: Roles Guard (Runs After JwtAuthGuard)

**What:** Checks if the authenticated user has the required role for the endpoint.
**When to use:** On write endpoints that require ADMIN role.

```typescript
// Source: NestJS official docs
// roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '../../common/types/role.enum';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]): ReturnType<typeof SetMetadata> =>
  SetMetadata(ROLES_KEY, roles);

// roles.guard.ts
import { CanActivate, ExecutionContext, ForbiddenException, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';
import { Role } from '../../common/types/role.enum';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true; // No roles required = any authenticated user
    const { user } = context.switchToHttp().getRequest();
    const hasRole = requiredRoles.includes(user.role);
    if (!hasRole) {
      throw new ForbiddenException('Permisos insuficientes');
    }
    return true;
  }
}
```

### Pattern 4: Guard Registration Order (Critical)

**What:** JwtAuthGuard MUST run before RolesGuard. With APP_GUARD, registration order determines execution order.
**When to use:** In AuthModule or AppModule providers.

```typescript
// Source: NestJS docs + github.com/nestjs/nest#5598
// auth.module.ts — register guards in correct order
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,   // FIRST: validates JWT, attaches user to request
  },
  {
    provide: APP_GUARD,
    useClass: RolesGuard,     // SECOND: reads user.role from request (set by JwtAuthGuard)
  },
],
```

**Critical:** If RolesGuard runs before JwtAuthGuard, `request.user` is undefined and role checks fail silently or throw. The order of `APP_GUARD` providers in the array determines execution order. JwtAuthGuard must be registered first.

### Pattern 5: JWT Strategy with Whitelist Check

**What:** Passport JWT strategy validates the token AND checks the email whitelist on every request.
**When to use:** Automatically invoked by JwtAuthGuard on every authenticated request.

```typescript
// Source: NestJS Passport docs + project CONTEXT.md decisions
// jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { UsersService } from '../../users/users.service';

interface JwtPayload {
  sub: number;
  email: string;
  role: string;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    configService: ConfigService,
    private readonly usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.getOrThrow<string>('JWT_SECRET'),
    });
  }

  async validate(payload: JwtPayload): Promise<{ id: number; email: string; role: string }> {
    // Whitelist check on EVERY request (per user decision)
    const user = await this.usersService.findActiveByEmail(payload.email);
    if (!user) {
      throw new UnauthorizedException('Usuario no autorizado');
    }
    return { id: user.id, email: user.email, role: user.role };
  }
}
```

### Pattern 6: Next.js 16 Proxy for Route Protection

**What:** `proxy.ts` replaces `middleware.ts` in Next.js 16. Redirects unauthenticated users to `/login`.
**When to use:** As the outermost route protection layer on the frontend.

```typescript
// Source: Next.js 16 official docs (nextjs.org/docs/app/api-reference/file-conventions/proxy)
// proxy.ts (project root or src/)
import { auth } from '@/auth';

export const proxy = auth((req) => {
  if (!req.auth && req.nextUrl.pathname !== '/login' && req.nextUrl.pathname !== '/acceso-denegado') {
    const loginUrl = new URL('/login', req.nextUrl.origin);
    loginUrl.searchParams.set('callbackUrl', req.nextUrl.pathname);
    return Response.redirect(loginUrl);
  }
});

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico|fonts|images).*)',
  ],
};
```

### Anti-Patterns to Avoid

- **Sending NextAuth session cookie to backend:** The NextAuth cookie is encrypted with `AUTH_SECRET` and cannot be verified by the NestJS backend. Always extract the backend JWT from the session and send it as `Authorization: Bearer`.
- **Sending Google `access_token` to backend:** Google access tokens are opaque and require a Google API call to verify. Use `id_token` (which is a signed JWT verifiable with Google's public keys).
- **Checking whitelist only at login:** If a user is removed from the whitelist, their JWT remains valid until expiry. Always check the whitelist in the JWT strategy's `validate()` method.
- **Using `autoLoadEntities: true` alongside manual DataSource:** The project uses a shared DataSource pattern. The User entity will be auto-discovered by the glob in `data-source.ts`.
- **Registering RolesGuard before JwtAuthGuard:** Guard execution follows registration order. RolesGuard needs `request.user` which is only set after JwtAuthGuard runs.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Google id_token verification | Manual JWKS fetch + JWT decode + signature verification | `google-auth-library` `OAuth2Client.verifyIdToken()` | Handles key rotation, caching, clock skew, and multiple signing keys automatically. |
| JWT signing/verification | Raw `jsonwebtoken` integration | `@nestjs/jwt` `JwtService` | Integrates with NestJS DI, ConfigService, and module system. |
| Passport JWT extraction | Manual `Authorization` header parsing | `passport-jwt` `ExtractJwt.fromAuthHeaderAsBearerToken()` | Handles edge cases: missing header, malformed bearer, whitespace. |
| OAuth flow | Manual Google OAuth redirect/callback implementation | Auth.js v5 Google provider | Handles CSRF, PKCE, state verification, token exchange, error recovery. |
| Session management | Custom cookie encryption + storage | Auth.js v5 JWT session strategy | Handles HTTP-only secure cookies, encryption with `AUTH_SECRET`, automatic rotation. |
| Route protection (frontend) | Custom fetch interceptors checking auth state | `proxy.ts` with `auth()` | Runs before route rendering, handles redirects server-side, works with streaming. |

**Key insight:** Authentication is a solved problem with many sharp edges (token rotation, CSRF, timing attacks, cookie security). Every hand-rolled component is a potential vulnerability.

## Common Pitfalls

### Pitfall 1: Three-Token Confusion
**What goes wrong:** Developers conflate Google's `id_token`, Google's `access_token`, NextAuth's session JWT, and the backend-minted JWT. Using the wrong token at the wrong point causes 401s.
**Why it happens:** Each OAuth library uses the word "token" for different things. NextAuth also calls its session cookie a "JWT" even though it's unrelated to the backend JWT.
**How to avoid:** Clearly name variables: `googleIdToken`, `backendAccessToken`, `nextAuthSessionToken`. In the jwt callback, only use `account.id_token`. In API calls, only use `session.accessToken` (which holds the backend JWT).
**Warning signs:** Frontend shows user as logged in but all API calls return 401. Session object has no `accessToken` field.

### Pitfall 2: signIn and jwt Callbacks Calling Backend Twice
**What goes wrong:** Both the `signIn` callback and `jwt` callback call `POST /api/auth/google`, resulting in two token exchange calls per login.
**Why it happens:** `signIn` callback needs to validate whitelist access (return false if not whitelisted). `jwt` callback needs to store the backend JWT. Both need the backend response.
**How to avoid:** Use only the `jwt` callback for the backend call. For the `signIn` callback, either: (a) let it always return true and handle rejection in the `jwt` callback by not storing a token (then proxy.ts will redirect on next check), or (b) use the `signIn` callback to redirect to `/acceso-denegado` and the `jwt` callback to store the token. The simplest architecture: make the call in `jwt`, and in `signIn` callback just validate that the provider is google.
**Warning signs:** Backend logs show two `POST /auth/google` requests for every login.

### Pitfall 3: Guard Registration Order
**What goes wrong:** `RolesGuard` registered as `APP_GUARD` before `JwtAuthGuard` means it runs first. `request.user` is undefined, so role checks fail or grant unintended access.
**Why it happens:** NestJS executes global guards in registration order. The issue is documented in [nestjs/nest#5598](https://github.com/nestjs/nest/issues/5598).
**How to avoid:** Register `JwtAuthGuard` as the FIRST `APP_GUARD` provider, `RolesGuard` as the SECOND. Verify with a test: call a `@Roles(Role.ADMIN)` endpoint with a USER token and confirm 403.
**Warning signs:** All role-protected endpoints return 500 or allow any authenticated user.

### Pitfall 4: Whitelist Checked Only at Login
**What goes wrong:** User removed from whitelist can still call API for the JWT lifetime (7 days).
**Why it happens:** Common OAuth pattern is: "check authorization at login time." But the JWT is a bearer credential independent of the whitelist state.
**How to avoid:** The `JwtStrategy.validate()` method queries the `users` table on every request. If the user is not found or `isActive` is false, throw `UnauthorizedException`. At 2-3 users, this is a trivial indexed query.
**Warning signs:** Deactivated user can still make authenticated API calls.

### Pitfall 5: AUTH_SECRET vs JWT_SECRET Confusion
**What goes wrong:** `AUTH_SECRET` (used by NextAuth to encrypt its session cookie) is confused with `JWT_SECRET` (used by NestJS to sign backend JWTs). Using the same secret for both, or mixing them up, causes verification failures.
**Why it happens:** Both are "secrets" related to "auth." NextAuth v5 auto-reads `AUTH_SECRET` env var.
**How to avoid:** Use distinct env vars: `AUTH_SECRET` for frontend (NextAuth), `JWT_SECRET` for backend (NestJS). Document both in `.env.example` files. The two secrets should be different values.
**Warning signs:** JWT verification fails after env changes. NextAuth session decryption errors.

### Pitfall 6: NextAuth Proxy Config Missing API Routes
**What goes wrong:** The `proxy.ts` matcher doesn't exclude `/api/auth/*` routes, causing NextAuth's route handler to be intercepted by the proxy and redirected to login.
**Why it happens:** Overly broad matcher pattern.
**How to avoid:** Ensure the matcher excludes `api` routes: `'/((?!api|_next/static|_next/image|favicon.ico).*)'`. The NextAuth route handler at `/api/auth/[...nextauth]` must be accessible without authentication.
**Warning signs:** Google OAuth callback redirects to login instead of completing authentication.

### Pitfall 7: TypeORM Migration for Seed Data
**What goes wrong:** Using `synchronize` or manual SQL to seed the first admin user instead of a migration. The seed is not reproducible across environments.
**Why it happens:** Developers conflate schema migrations with data seeding.
**How to avoid:** Create a TypeORM migration that both creates the `users` table and inserts the seed admin user. The `up()` method runs `INSERT INTO users (email, role, name) VALUES ('admin@email.com', 'admin', 'Admin')`. This runs automatically via `migrationsRun: true`.
**Warning signs:** App works locally but production has no admin user. Test environment has no users.

## Code Examples

### User Entity

```typescript
// Source: Project BaseEntity pattern + CONTEXT.md decisions
// users/entities/user.entity.ts
import { Column, Entity, Unique } from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';
import { Role } from '../../common/types/role.enum';

@Entity('users')
@Unique(['email'])
export class User extends BaseEntity {
  @Column({ type: 'varchar', length: 255 })
  email!: string;

  @Column({ type: 'varchar', length: 255, nullable: true })
  name!: string | null;

  @Column({ type: 'varchar', length: 500, nullable: true, name: 'picture_url' })
  pictureUrl!: string | null;

  @Column({ type: 'varchar', length: 50, name: 'google_id', nullable: true })
  googleId!: string | null;

  @Column({ type: 'enum', enum: Role, default: Role.USER })
  role!: Role;

  @Column({ type: 'boolean', name: 'is_active', default: true })
  isActive!: boolean;
}
```

### Auth Service (Backend Token Exchange)

```typescript
// Source: google-auth-library official docs + NestJS JWT docs
// auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { JwtService } from '@nestjs/jwt';
import { OAuth2Client } from 'google-auth-library';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  private readonly googleClient: OAuth2Client;

  constructor(
    private readonly jwtService: JwtService,
    private readonly usersService: UsersService,
    private readonly configService: ConfigService,
  ) {
    this.googleClient = new OAuth2Client(
      this.configService.getOrThrow<string>('GOOGLE_CLIENT_ID'),
    );
  }

  async validateGoogleToken(idToken: string): Promise<{
    accessToken: string;
    user: { id: number; email: string; role: string; name: string | null; pictureUrl: string | null };
  }> {
    // 1. Verify Google id_token
    const ticket = await this.googleClient.verifyIdToken({
      idToken,
      audience: this.configService.getOrThrow<string>('GOOGLE_CLIENT_ID'),
    });
    const payload = ticket.getPayload();
    if (!payload?.email) {
      throw new UnauthorizedException('Token de Google invalido');
    }

    // 2. Check whitelist
    const user = await this.usersService.findActiveByEmail(payload.email);
    if (!user) {
      throw new UnauthorizedException('Usuario no autorizado');
    }

    // 3. Update profile data from Google (name, picture)
    await this.usersService.updateGoogleProfile(user.id, {
      name: payload.name ?? null,
      pictureUrl: payload.picture ?? null,
      googleId: payload.sub,
    });

    // 4. Mint backend JWT
    const accessToken = this.jwtService.sign({
      sub: user.id,
      email: user.email,
      role: user.role,
    });

    return {
      accessToken,
      user: {
        id: user.id,
        email: user.email,
        role: user.role,
        name: payload.name ?? user.name,
        pictureUrl: payload.picture ?? user.pictureUrl,
      },
    };
  }
}
```

### AuthModule Configuration

```typescript
// Source: NestJS official docs
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { APP_GUARD } from '@nestjs/core';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { JwtStrategy } from './strategies/jwt.strategy';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { RolesGuard } from './guards/roles.guard';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        secret: configService.getOrThrow<string>('JWT_SECRET'),
        signOptions: { expiresIn: '7d' },
      }),
    }),
    UsersModule,
  ],
  controllers: [AuthController],
  providers: [
    AuthService,
    JwtStrategy,
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,   // MUST be first
    },
    {
      provide: APP_GUARD,
      useClass: RolesGuard,     // MUST be second
    },
  ],
  exports: [AuthService],
})
export class AuthModule {}
```

### TypeScript Module Augmentation (Frontend)

```typescript
// Source: Auth.js TypeScript docs (authjs.dev/getting-started/typescript)
// types/next-auth.d.ts
import 'next-auth';
import 'next-auth/jwt';

declare module 'next-auth' {
  interface Session {
    accessToken: string;
    user: {
      id: string;
      role: string;
      email: string;
      name: string;
      image: string;
    };
  }

  interface User {
    role?: string;
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    backendToken?: string;
    role?: string;
    userId?: string;
  }
}
```

### Authenticated API Fetch Wrapper

```typescript
// Source: Auth.js third-party backend guide
// lib/api.ts
import { auth } from '@/auth';

const API_URL = process.env.NEXT_PUBLIC_API_URL;

export async function apiFetch<T>(
  path: string,
  options: RequestInit = {},
): Promise<T> {
  const session = await auth();
  if (!session?.accessToken) {
    throw new Error('No authenticated session');
  }

  const res = await fetch(`${API_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${session.accessToken}`,
      ...options.headers,
    },
  });

  if (!res.ok) {
    throw new Error(`API error: ${res.status}`);
  }

  return res.json() as Promise<T>;
}
```

### SessionProvider Client Wrapper

```typescript
// Source: Auth.js SessionProvider docs
// providers/SessionProvider.tsx
'use client';

import { SessionProvider as NextAuthSessionProvider } from 'next-auth/react';
import type { ReactElement, ReactNode } from 'react';

export function SessionProvider({
  children,
}: {
  children: ReactNode;
}): ReactElement {
  return <NextAuthSessionProvider>{children}</NextAuthSessionProvider>;
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `middleware.ts` for route protection | `proxy.ts` (Next.js 16) | Next.js 16.0.0 (2025) | File renamed, function export renamed from `middleware` to `proxy`. Config matcher unchanged. Node.js runtime only (no Edge). |
| `getServerSession(authOptions)` | `auth()` function from auth.ts | Auth.js v5 | Single unified function for server-side session access. No need to pass authOptions everywhere. |
| `next-auth@4` with pages router patterns | `next-auth@beta` (Auth.js v5) | Auth.js v5 beta (2024-present) | App Router native, `proxy.ts` integration, unified `auth()` function, `AUTH_*` env var auto-detection. |
| `@nestjs/jwt@10` / `@nestjs/passport@10` | `@nestjs/jwt@11` / `@nestjs/passport@11` | 2025 | Updated for NestJS 11 compatibility. No major API changes. |
| Manual Passport strategy wiring | `PassportModule.register({ defaultStrategy: 'jwt' })` | NestJS 11 | Simplified module registration. |

**Deprecated/outdated:**
- `middleware.ts` in Next.js 16: Renamed to `proxy.ts`. A codemod is available: `npx @next/codemod@canary middleware-to-proxy .`
- `NEXTAUTH_SECRET` env var: Auth.js v5 uses `AUTH_SECRET` (auto-detected). `NEXTAUTH_SECRET` still works as fallback but is deprecated.
- `getServerSession()`: Replaced by `auth()` in Auth.js v5.

## Open Questions

1. **Exact `next-auth@beta` version stability**
   - What we know: Auth.js v5 has been in beta since 2024. It's widely used in production despite the beta tag. The API is considered stable.
   - What's unclear: When stable v5 will release. Breaking changes between beta versions are possible.
   - Recommendation: Pin the version in `package.json` (no caret). Use `npm install next-auth@beta` and note the exact version installed. Test thoroughly.

2. **Double backend call in signIn + jwt callbacks**
   - What we know: Both callbacks fire during sign-in. If both call the backend, that's two requests.
   - What's unclear: Whether NextAuth guarantees the signIn callback completes before the jwt callback (ordering is documented but implementation details vary).
   - Recommendation: Make the backend call only in the `jwt` callback (when `account` is present). For the `signIn` callback, simply validate the provider is google and return true. If the backend call in `jwt` fails, the session won't have a `backendToken`, and `proxy.ts` will redirect to login on the next navigation.

3. **Swagger bearer token in Swagger UI**
   - What we know: Swagger is dev-only and already gated by NODE_ENV. The ADMIN will manage users via Swagger endpoints.
   - What's unclear: How to conveniently test authenticated endpoints in Swagger (adding `.addBearerAuth()` to Swagger config).
   - Recommendation: Add `.addBearerAuth()` to the Swagger `DocumentBuilder` and `@ApiBearerAuth()` decorator on authenticated controllers. This lets the ADMIN paste their JWT in Swagger's "Authorize" dialog.

## Sources

### Primary (HIGH confidence)
- [Auth.js official installation docs](https://authjs.dev/getting-started/installation) - Setup, providers, route handler
- [Auth.js Google provider docs](https://authjs.dev/getting-started/providers/google) - Google OAuth configuration
- [Auth.js session management docs](https://authjs.dev/getting-started/session-management/protecting) - Route protection, proxy pattern
- [Auth.js TypeScript docs](https://authjs.dev/getting-started/typescript) - Module augmentation patterns
- [Auth.js third-party backend guide](https://authjs.dev/guides/integrating-third-party-backends) - Token exchange with external backend
- [Next.js 16 proxy.ts docs](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) - Proxy file convention, matcher, runtime
- [NestJS Authentication docs](https://docs.nestjs.com/security/authentication) - JWT, Passport, guards
- [NestJS Guards docs](https://docs.nestjs.com/guards) - Guard execution, APP_GUARD, roles
- [Google ID token verification guide](https://developers.google.com/identity/gsi/web/guides/verify-google-id-token) - google-auth-library usage
- [@nestjs/jwt npm](https://www.npmjs.com/package/@nestjs/jwt) - Version 11.0.2
- [@nestjs/passport npm](https://www.npmjs.com/package/@nestjs/passport) - Version 11.0.5

### Secondary (MEDIUM confidence)
- [NestJS guard execution order issue](https://github.com/nestjs/nest/issues/5598) - APP_GUARD ordering confirmed by maintainers
- [NextAuth + custom backend discussion](https://github.com/nextauthjs/next-auth/discussions/8884) - Token exchange pattern with NestJS
- [NextAuth whitelist discussion](https://github.com/nextauthjs/next-auth/discussions/4991) - signIn callback for access control
- [NestJS guards authorization guide (2026)](https://oneuptime.com/blog/post/2026-02-02-nestjs-guards-authorization/view) - Global guards + roles pattern

### Tertiary (LOW confidence)
- [Auth.js v5 beta stability](https://github.com/nextauthjs/next-auth/discussions/8487) - Community discussion on production readiness. Beta tag persists but API is considered stable.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All libraries are official/standard for their ecosystems. Versions verified on npm.
- Architecture: HIGH - Token exchange pattern is well-documented in Auth.js official guide and NestJS docs. Guard patterns are standard NestJS.
- Pitfalls: HIGH - All major pitfalls verified via official docs, GitHub issues, or multiple community sources. The three-token confusion and guard ordering issues are particularly well-documented.

**Research date:** 2026-03-01
**Valid until:** 2026-04-01 (30 days - stack is stable, Auth.js v5 beta API unlikely to change significantly)
