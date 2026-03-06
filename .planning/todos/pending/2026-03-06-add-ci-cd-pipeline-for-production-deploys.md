---
created: 2026-03-06T22:20:58.421Z
title: Add CI/CD pipeline for production deploys
area: tooling
files:
  - .github/workflows/
  - nemea-front/.github/workflows/
  - nemea-back/.github/workflows/
---

## Problem

No hay CI/CD configurado. Los deploys son manuales y no hay validación automática antes de mergear PRs. Cuando el proyecto esté en producción (Railway + Vercel) se necesita un pipeline que valide builds, tests, y lint antes de cada merge, y que automatice deploys.

## Solution

- GitHub Actions para ambos sub-repos:
  - On PR to development: lint + build + test (gate de calidad)
  - On merge to main (tag [PROD]): deploy automático
- nemea-back: build → test → deploy a Railway
- nemea-front: build → deploy a Vercel (o usar Vercel's built-in GitHub integration)
- Considerar: matrix testing (Node versions), caching de node_modules, secrets management para env vars de producción
