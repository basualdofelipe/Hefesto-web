# Technology Stack

**Analysis Date:** 2026-02-28

## Languages

**Primary:**
- TypeScript 5.x - Strict mode enabled across all projects
- JavaScript (ES2017) - Build outputs and config files

**Templating/Markup:**
- JSX/TSX - React components with TypeScript support
- CSS 4 - Tailwind CSS v4 for styling

## Runtime

**Environment:**
- Node.js 20+ (inferred from package.json `@types/node: ^20`)
- npm (package manager)

**Package Manager:**
- npm - Lockfile: `package-lock.json` present

## Frameworks

**Frontend (nemea-front/):**
- Next.js 16.1.6 - Full-stack React with App Router
  - nextAuth v4+ planned for OAuth integration (Google)
  - `next-themes` 0.4.6 - Theme management (light/dark)

**Backend (nemea-back/):**
- NestJS - Scaffolding phase, not yet implemented
- TypeORM - ORM for PostgreSQL (planned)

**UI/Component Library:**
- React 19.2.3 - JavaScript library for building UIs
- React DOM 19.2.3 - DOM rendering
- Shadcn/ui - Headless component library
  - radix-ui 1.4.3 - Unstyled, accessible components
  - class-variance-authority 0.7.1 - CSS-in-JS variant system
  - clsx 2.1.1 - Utility for conditional class names
  - tailwind-merge 3.5.0 - Smart class merging
  - lucide-react 0.575.0 - Icon library
  - sonner 2.0.7 - Toast notifications

## Styling & CSS

**Framework:**
- Tailwind CSS v4 - Utility-first CSS framework
  - @tailwindcss/postcss 4.x - PostCSS plugin for Tailwind
  - tailwindcss-animate 1.0.7 - Animation utilities
  - prettier-plugin-tailwindcss 0.7.2 - Auto-sorts class names

**Processing:**
- PostCSS 8.5.6 - CSS transformation
- Autoprefixer 10.4.27 - Vendor prefix handling

**Typography:**
- Poppins (Google Fonts) - Body text, weights: 300, 400, 500, 600
- JetBrains Mono (Google Fonts) - Monospace, weights: 400, 500
- EngravingCC (local font) - Custom display font (Nemea branding)
- EngravingShadedCC (local font) - Custom shaded display variant

## Form Management

**Library:**
- react-hook-form 7.71.2 - Form state management
  - @hookform/resolvers 5.2.2 - Validation adapters
  - zod 4.3.6 - Schema validation and TypeScript inference

## Development Tools

**Code Quality:**
- ESLint 9.x - Linting
  - @typescript-eslint/parser 8.56.1 - TypeScript parsing
  - @typescript-eslint/eslint-plugin 8.56.1 - TypeScript rules
  - typescript-eslint 8.56.1 - Combined ESLint config
  - eslint-config-next 16.1.6 - Next.js recommended rules
  - eslint-plugin-sonarjs 4.0.0 - Code quality rules (cognitive complexity limit: 30)

**Formatting:**
- Prettier 3.8.1 - Code formatter
  - prettier-plugin-tailwindcss 0.7.2 - Tailwind class sorting
  - Configuration: Single quotes, 2 spaces, trailing commas, auto line ending

**Testing:**
- Jest 30.2.0 - Test runner
  - jest-environment-jsdom 30.2.0 - DOM testing environment
  - ts-jest 29.4.6 - TypeScript support for Jest
  - @testing-library/react 16.3.2 - React testing utilities
  - @testing-library/dom 10.4.1 - DOM testing utilities
  - @testing-library/jest-dom 6.9.1 - Jest matchers for DOM
  - @testing-library/user-event 14.6.1 - User interaction simulation
  - @types/jest 30.0.0 - TypeScript types

**Build & Bundling:**
- Webpack (via Next.js) - Module bundler
  - @svgr/webpack 8.1.0 - SVG to React components
- Turbopack (Next.js v16) - Next-gen build system enabled in config

**Git Hooks:**
- Husky 9.1.7 - Pre-commit hooks (configured in package.json)

## Configuration

**Environment:**
- `.env.example` present at `nemea-front/.env.example`
- Environment variables configured in next.config.mjs:
  - `NEXT_PUBLIC_APP_NAME` - "Nemea"
  - `NEXT_PUBLIC_API_URL` - Backend API endpoint (dev: `http://localhost:4000`)
  - `NEXTAUTH_URL` - NextAuth callback URL (dev: `http://localhost:3000`)
  - `NEXTAUTH_SECRET` - Session encryption key
  - `NODE_ENV` - Environment mode (development/production)

**Build Configuration:**
- `next.config.mjs` - Next.js build settings
  - Removes console logs in production (except error/warn)
  - React Strict Mode enabled
  - Turbopack enabled
  - SVG as React components via @svgr/webpack
  - Watch mode optimizations for development

**TypeScript:**
- `tsconfig.json` configured with:
  - Target: ES2017
  - Strict mode enabled
  - Module resolution: bundler
  - Path alias: `@/*` → `./src/*`

## Database

**Primary (Backend):**
- PostgreSQL (hosted on Railway in production, Docker Compose locally)
- Planned ORM: TypeORM with migrations (no synchronize in any environment)

## Deployment Targets

**Frontend:**
- Vercel - Automated deployment from GitHub

**Backend:**
- Railway - ~$5/month, includes PostgreSQL hosting

## Platform Requirements

**Development:**
- Node.js 20+
- npm 9+
- Git 2.x
- Docker Desktop (for local PostgreSQL via Docker Compose)

**Production:**
- Vercel (frontend) - No additional requirements
- Railway (backend + database) - PostgreSQL hosted

---

*Stack analysis: 2026-02-28*
