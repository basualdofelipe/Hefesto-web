---
created: 2026-03-06T18:53:00.000Z
title: Dark/light mode toggle in sidebar or header
area: ui
files:
  - nemea-front/src/components/layout/AppSidebar.tsx
  - nemea-front/src/components/layout/Header.tsx
---

## Problem

No hay forma de cambiar entre modo oscuro y claro desde la UI. El toggle debería estar siempre accesible, ya sea en el sidebar (abajo a la izquierda) o en el header (al lado del avatar del usuario). Tiene que ser un control permanente, no escondido en settings.

## Solution

- Agregar un toggle de dark/light mode con ícono (sol/luna o similar)
- Ubicación: sidebar footer (abajo a la izquierda) o header (al lado del usuario) — decidir cuál queda mejor
- Usar next-themes para persistir la preferencia
- Respetar la preferencia del sistema como default
- El toggle debe estar visible en todas las páginas, siempre accesible
