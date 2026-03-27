---
created: 2026-03-26T00:00:00.000Z
title: Role-based redirect on login — investors land on /dashboard
area: auth
files:
  - nemea-front/src/proxy.ts
  - nemea-front/src/app/(app)/page.tsx
---

## Problem

Home page (`/`) is a placeholder with a theme toggle. When an investor logs in, they see a blank page instead of useful content. Admin also sees the placeholder instead of something relevant.

## Solution

- In middleware/proxy: after auth, check role. USER → redirect to `/dashboard`. ADMIN → either `/productos` or a future admin home.
- Or: make the home page smart — render different content per role.
- Scope to Phase 8 (hardening) or Phase 13 (dashboard), depending on when `/dashboard` route exists.
