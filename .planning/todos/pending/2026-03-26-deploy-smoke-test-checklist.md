---
created: 2026-03-26T00:00:00.000Z
title: Add deploy smoke test to phase verification checklist
area: tooling
files: []
---

## Problem

v1.0 had a CORS bug in production (trailing slash in FRONTEND_URL) that wasn't caught because there's no post-deploy verification. Each phase that touches backend (migrations, new endpoints) should verify production works.

## Solution

Add to each backend-touching phase verification:
1. Railway deploy succeeded (check dashboard)
2. `GET /api/health` returns 200
3. New endpoints respond correctly (not 404/500)
4. CORS still works (try a mutation from the frontend)
5. Migrations ran (check new tables/columns exist)

Not a separate phase — a checklist item in every phase that deploys backend changes.
