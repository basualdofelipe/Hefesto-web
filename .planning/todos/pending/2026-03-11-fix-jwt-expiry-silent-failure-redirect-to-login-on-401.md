---
created: 2026-03-11T16:40:19.631Z
title: Fix JWT expiry silent failure - redirect to login on 401
area: auth
files:
  - nemea-front/src/lib/apiClient.ts
  - nemea-front/src/lib/api.ts
---

## Problem

Cuando el JWT del backend expira, la app no redirige al login automáticamente. En cambio, la UI muestra errores (fetches que fallan con 401) sin que el usuario entienda qué pasó. El usuario queda "logueado" en NextAuth pero con un token de backend inválido.

## Solution

Agregar un interceptor global de 401 en `apiClient.ts` (y posiblemente `api.ts`): cuando cualquier llamada al backend retorna 401, llamar a `signOut()` de NextAuth con `callbackUrl: '/login'`. Esto cierra la sesión del lado del cliente y redirige al login limpiamente.

Considerar también manejar el refresh del token si se quiere evitar logout en cada expiración (depende de si el backend implementa refresh tokens).
