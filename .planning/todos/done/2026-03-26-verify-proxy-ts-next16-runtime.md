---
created: 2026-03-26T00:00:00.000Z
title: Verify proxy.ts works at runtime in Next.js 16 before Phase 8
area: auth
files:
  - nemea-front/src/proxy.ts
---

## Problem

Research conflict: STACK.md says `proxy.ts` is the correct filename for Next.js 16 middleware. The senior review says it's dead code because there's no `middleware.ts`. Need runtime verification before spending effort on either approach.

## Solution

1. Add `console.log('PROXY HIT')` to proxy.ts
2. Run `npm run dev` in nemea-front
3. Navigate to any app route
4. If log appears → proxy.ts works, no rename needed
5. If no log → create `middleware.ts` that re-exports from proxy.ts

Do this at the START of Phase 8 before planning.
