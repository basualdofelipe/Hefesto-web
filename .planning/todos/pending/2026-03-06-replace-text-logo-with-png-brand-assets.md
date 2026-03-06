---
created: 2026-03-06T18:55:00.000Z
title: Replace text logo with PNG brand assets
area: ui
files:
  - nemea-front/src/components/layout/AppSidebar.tsx
  - nemea-front/src/app/(auth)/login/page.tsx
  - nemea-front/public/brand/logo.png
  - nemea-front/public/brand/Isotipo.png
  - nemea-front/public/brand/leon beige.png
  - nemea-front/public/brand/leon blanco.png
  - nemea-front/public/brand/leon dorado.png
---

## Problem

El sidebar muestra "NEMEA" como texto plano con una fuente serif. El usuario tiene assets de marca reales en `public/brand/`:
- `logo.png` — logo completo "NEMEA" en dorado/terracota (serif con bordes)
- `Isotipo.png` — isotipo (probablemente el león)
- `leon beige.png`, `leon blanco.png`, `leon dorado.png` — variantes del león/isotipo

Hay que reemplazar el texto por las imágenes reales y agregar más presencia de marca (isotipo en más lugares).

## Solution

- Sidebar: reemplazar el `<span>NEMEA</span>` por `<Image src="/brand/logo.png">` con tamaño adecuado
- Login page: usar el logo real en vez de texto
- Agregar isotipo (león) en lugares estratégicos: favicon, sidebar colapsado, loading states
- Considerar variante blanca/beige del león para dark mode
- Verificar que las imágenes tengan fondo transparente para ambos modos
