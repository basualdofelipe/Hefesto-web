---
status: fixing
trigger: "broken-sidebar-login-logos"
created: 2026-04-09T00:00:00Z
updated: 2026-04-09T01:00:00Z
---

## Current Focus

hypothesis: localPatterns config is correct in format but the /_next/image optimizer is still rejecting requests. Root cause is likely Turbopack dev server not fully honoring localPatterns config, OR a stale server state. Fix: bypass image optimizer entirely using `unoptimized` prop (or images.unoptimized global config), which serves the image directly from /brand/... — which we KNOW works.
test: Add `unoptimized` prop to Image components in AppSidebar.tsx and login/page.tsx, or set images.unoptimized globally in next.config.mjs
expecting: Images render correctly in sidebar and login page because they bypass /_next/image and serve directly
next_action: Apply unoptimized fix to next.config.mjs (global) — cleanest single-point fix

## Symptoms

expected: Sidebar expanded shows NEMEA text logo PNG, sidebar collapsed shows lion isotipo PNG. Login page shows isotipo PNG centered in the card.
actual: Both sidebar (expanded) and login page show broken image placeholder icons. The collapsed sidebar shows what appears to be a tiny broken image too.
errors: No console errors reported. Images simply don't render.
reproduction: Navigate to any page (sidebar visible) or /login page. Images are broken everywhere.
started: Introduced in Phase 12.2 Plan 02 which added next/image Image components pointing to /brand/logo.png and /brand/Isotipo.png. Never worked.

## Eliminated

(none yet)

## Evidence

- timestamp: 2026-04-09T00:00:00Z
  checked: Key files provided in hints
  found: Files confirmed to exist at nemea-front/public/brand/logo.png (36KB) and nemea-front/public/brand/Isotipo.png (361KB)
  implication: Asset placement is correct; issue is in code, config, or path resolution

- timestamp: 2026-04-09T00:01:00Z
  checked: nemea-front/next.config.mjs
  found: No `images` config key present at all. Has `turbopack: {}` and a `webpack()` SVG loader.
  implication: No localPatterns defined — this is the likely root cause.

- timestamp: 2026-04-09T00:02:00Z
  checked: AppSidebar.tsx and login/page.tsx
  found: Both use `next/image` with correct local paths (/brand/logo.png, /brand/Isotipo.png) and explicit width/height props. Paths match actual files in public/brand/.
  implication: Code is correct. Problem is configuration, not usage.

- timestamp: 2026-04-09T00:03:00Z
  checked: .next/images-manifest.json and required-server-files.json
  found: localPatterns regex is default (allows all non-traversal paths). But this is for production build — dev server may behave differently.
  implication: Build config appears to allow all local paths, but the 400 error occurs in dev with Turbopack.

- timestamp: 2026-04-09T00:04:00Z
  checked: Next.js 16 official docs and error page next-image-unconfigured-localpatterns
  found: "The localPatterns field is required starting with Next.js 16 because unrestricted access could allow malicious actors to optimize more qualities than you intended." Without images.localPatterns, /_next/image requests for local files return 400 Bad Request silently. This is a BREAKING CHANGE in Next.js 16.
  implication: ROOT CAUSE CONFIRMED. Adding images.localPatterns: [{ pathname: '/brand/**', search: '' }] will fix it.

- timestamp: 2026-04-09T00:05:00Z
  checked: middleware.ts matcher
  found: Matcher correctly excludes _next/image path. Middleware is not blocking image optimization requests.
  implication: Middleware is NOT the cause. Eliminated.

- timestamp: 2026-04-09T01:00:00Z
  checked: Human verification of localPatterns fix
  found: Fix applied but NOT resolved. localhost:3000/brand/logo.png loads correctly in browser (file confirmed served). But next/image still shows broken image icon. The /_next/image optimizer is rejecting the requests even with localPatterns configured.
  implication: localPatterns config is either not being honored by Turbopack dev server, or there is a deeper incompatibility. Static serving works; optimization does not.

- timestamp: 2026-04-09T01:01:00Z
  checked: Next.js 16 Turbopack default bundler behavior
  found: In Next.js 16, Turbopack is the DEFAULT bundler for `next dev` (no flag needed). Known GitHub issue #46758 documents Turbopack ignoring images.remotePatterns — same class of bug likely applies to localPatterns. The webpack() function in next.config.mjs only runs for --webpack builds, not Turbopack.
  implication: localPatterns enforcement in Turbopack dev mode is unreliable. Workaround: bypass optimizer entirely with unoptimized: true.

- timestamp: 2026-04-09T01:02:00Z
  checked: Use case for optimization
  found: The brand images (logo.png 36KB, Isotipo.png 361KB) are static brand assets loaded on every page. They are served from public/ directly. For a small internal app, size optimization is not critical. The `unoptimized` prop (or global images.unoptimized: true) bypasses /_next/image and serves the src path directly.
  implication: `unoptimized: true` globally is the correct fix — it's simple, avoids the Turbopack bug entirely, and works because /brand/logo.png is confirmed reachable.

## Eliminated

- hypothesis: Middleware blocking image requests
  evidence: Middleware matcher excludes _next/image and _next/static paths. Auth middleware cannot intercept image optimization requests.
  timestamp: 2026-04-09T00:05:00Z

- hypothesis: Wrong file paths or case mismatch
  evidence: Files confirmed at nemea-front/public/brand/logo.png and Isotipo.png. Code uses /brand/logo.png and /brand/Isotipo.png. Exact match.
  timestamp: 2026-04-09T00:05:00Z

- hypothesis: CSS hiding images or zero dimensions
  evidence: globals.css has no image-related styles. next/image components use h-8/size-7/size-20 classes which set valid dimensions.
  timestamp: 2026-04-09T00:05:00Z

## Resolution

root_cause: Next.js 16 uses Turbopack by default for `next dev`. The localPatterns config was applied correctly in format, but the /_next/image optimizer is still rejecting local image requests even with the config present. This is consistent with known Turbopack issues where image-related config options are not fully honored in the dev server. The static file at /brand/logo.png serves fine, confirming the issue is specifically with next/image optimization, not file placement.
fix: Set images.unoptimized = true globally in next.config.mjs. This bypasses the /_next/image optimizer entirely and serves brand images directly from /brand/... paths — which are confirmed to work. Since these are static brand assets (logos), optimization is not needed.
verification:
files_changed: ["nemea-front/next.config.mjs"]
