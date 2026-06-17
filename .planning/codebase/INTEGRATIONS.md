# External Integrations

**Analysis Date:** 2026-02-28

## APIs & External Services

**Authentication:**
- Google OAuth - User identity and authentication
  - SDK/Client: NextAuth v4+ (to be implemented)
  - Auth flow: Google OAuth → NextAuth (frontend) → JWT validation (backend)
  - Whitelist model: Emails stored in `users` table, admin-managed (no public registration)

**Business Integration:**
- Tiendanube API - E-commerce platform integration (planned for V1)
  - Purpose: Fetch commission rates, payment methods, installment plans
  - Configuration: Stored in `config_tiendanube` table
  - Data needed: Payment method rates, installment rates, IIBB rates, IVA

## Data Storage

**Primary Database:**
- PostgreSQL (Railway-hosted in production)
  - Connection env var: `DATABASE_URL` (not yet configured in .env.example)
  - Client: TypeORM (NestJS ORM)
  - Migration strategy: Always use migrations in dev and prod (no synchronize)
  - Docker Compose: Local PostgreSQL container for development

**File Storage:**
- Local filesystem only
  - Public assets: `hefesto-front/public/`
    - Custom fonts: `fonts/EngravingCC.ttf`, `fonts/EngravingShadedCC.ttf`
    - Other static files: icons, images (managed by Next.js)
  - No external cloud storage (S3, Cloudinary, etc.) planned for V1

**Caching:**
- None currently
  - React Query / SWR not in dependencies (could be added for client-side caching)
  - Backend caching: Not configured yet

## Authentication & Identity

**Auth Provider:**
- Google OAuth via NextAuth (frontend) + JWT (backend)

**Implementation Details:**
- Frontend: NextAuth 4+ (not yet installed)
  - Configuration location: To be implemented in `hefesto-front/src/auth/`
  - Callback URL: `http://localhost:3000/api/auth/callback/google` (dev)
  - Redirect on login: `/productos` (or protected route)

- Backend: JWT validation (NestJS AuthModule planned)
  - Endpoints for validation: `POST /auth/validate`
  - Token source: NextAuth JWT passed in `Authorization: Bearer` header
  - User whitelist: `users` table with email and role (ADMIN / USER)
  - No password management: Auth entirely delegated to Google

## Monitoring & Observability

**Error Tracking:**
- Not configured
  - Could use: Sentry (recommended for production)

**Logs:**
- Development: Console output via `next dev` and NestJS logger
- Production: Vercel/Railway built-in logging
- No external service configured yet

**Environment-specific logging:**
- `next.config.mjs` removes console logs in production (except error/warn)

## CI/CD & Deployment

**Hosting:**

**Frontend:**
- Vercel - Free tier
  - Source: GitHub (`Hefesto-web/hefesto-front` development branch)
  - Deployment trigger: Push to `main` branch (production) and `development` branch (staging)
  - Environment variables: Set in Vercel dashboard
  - Build command: `npm run build`
  - Start command: `npm run start`

**Backend:**
- Railway - ~$5/month
  - Source: GitHub (`Hefesto-web/hefesto-back` development branch)
  - Deployment trigger: Push to `main` branch (production) and `development` (staging)
  - Includes PostgreSQL hosting
  - Environment variables: Set in Railway dashboard

**CI Pipeline:**
- Not yet implemented
  - Could use: GitHub Actions for testing before merge
  - Pre-commit: Husky hooks configured (lint, prettier)

## Environment Configuration

**Required environment variables (Development):**

**Frontend (.env.local or .env.development):**
```
NEXT_PUBLIC_APP_NAME=Hefesto
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=<generated-with-openssl>
NODE_ENV=development
```

**Frontend (Production on Vercel):**
```
NEXT_PUBLIC_APP_NAME=Hefesto
NEXT_PUBLIC_API_URL=https://hefesto-api.railway.app  # or Railway URL
NEXTAUTH_URL=https://hefesto.vercel.app
NEXTAUTH_SECRET=<production-secret>
NODE_ENV=production
```

**Backend (Development - not yet scaffolded):**
```
DATABASE_URL=postgresql://user:password@localhost:5432/hefesto_dev
NODE_ENV=development
JWT_SECRET=<secret-for-token-signing>
GOOGLE_CLIENT_ID=<from-google-console>
GOOGLE_CLIENT_SECRET=<from-google-console>
```

**Backend (Production on Railway):**
```
DATABASE_URL=<railway-postgresql-url>
NODE_ENV=production
JWT_SECRET=<production-secret>
DATABASE_SSL=true
```

**Secrets location:**
- Development: Local `.env.local` files (not committed)
- Production: Vercel dashboard (frontend), Railway dashboard (backend)
- `.env.example` files present for template reference (`hefesto-front/.env.example`)

## Webhooks & Callbacks

**Incoming:**
- NextAuth OAuth callback: `POST /api/auth/callback/google`
  - Path: `hefesto-front/src/app/api/auth/callback/google/route.ts` (to be implemented)
  - Triggered by: Google OAuth after user consent
  - Purpose: Exchange auth code for JWT session

**Outgoing:**
- Backend to frontend: No webhooks
- Tiendanube webhooks: Not implemented yet
  - Could be added in V2 for order status updates

## Integration Timeline

**V1 (Current & Planned):**
- Auth: Google OAuth + NextAuth + JWT (Phase 3)
- Database: PostgreSQL on Railway (Phases 2 onwards)
- Deployment: Vercel (frontend), Railway (backend)
- Config: Tiendanube rates storage (Phase 6)

**V2 (Future):**
- Tiendanube Order Webhooks: For B2B sales tracking
- Error Tracking Service: Sentry for monitoring
- Client-side caching: React Query or SWR
- Data export: CSV/Excel generation

## Service Endpoints (Development)

**Frontend:**
- Local: `http://localhost:3000`
- Vercel staging: To be set up

**Backend:**
- Local: `http://localhost:4000`
- Railway staging: To be set up

**Database:**
- Local: `postgresql://postgres:password@localhost:5432/hefesto_dev`
- Railway production: Configured via `DATABASE_URL` env var

---

*Integration audit: 2026-02-28*
