---
phase: quick-260527-usx
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - nemea-front/src/app/(auth)/login/page.tsx
autonomous: true
requirements: [QUICK-LOGO]
must_haves:
  truths:
    - "Logo del leon en /login se ve al doble de tamano que antes"
    - "Logo esta horizontalmente centrado en el CardHeader"
    - "Ningun otro asset (sidebar logo, texto, botones) se modifico"
  artifacts:
    - path: "nemea-front/src/app/(auth)/login/page.tsx"
      provides: "Login page con logo 2x y centrado"
      contains: "size-40"
  key_links:
    - from: "CardHeader"
      to: "Image (logo)"
      via: "justify-items-center sobre el grid del CardHeader"
      pattern: "justify-items-center"
---

<objective>
Agrandar 2x el logo del leon en la pagina de login y centrarlo horizontalmente.

Purpose: El usuario lo quiere "el doble de grande y centrado". El logo se ve chico y desplazado a la izquierda.
Output: `nemea-front/src/app/(auth)/login/page.tsx` con el `<Image>` del logo a 160x160 / `size-40` y centrado.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md

# Archivo a editar (unico)
@nemea-front/src/app/(auth)/login/page.tsx

<diagnosis>
ROOT CAUSE del logo desplazado a la izquierda:
El componente shadcn `CardHeader` (nemea-front/src/components/ui/card.tsx:23) usa `display: grid`.
En un grid, `items-center` solo afecta el eje de bloque (vertical), por eso la imagen queda
alineada a la izquierda. El texto descripcion parece centrado solo por `text-center`.
Para centrar horizontalmente un item de grid hace falta `justify-items-center` en el contenedor grid (CardHeader).

NO modificar `src/components/ui/card.tsx` (primitiva shadcn compartida). El fix queda local en la pagina de login.
</diagnosis>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Agrandar 2x y centrar el logo del login</name>
  <files>nemea-front/src/app/(auth)/login/page.tsx</files>
  <action>
En el `<CardHeader>` (linea ~55), agregar `justify-items-center` al className existente, quedando algo como `items-center justify-items-center space-y-2 text-center`. Esto centra horizontalmente el item de grid (la imagen). NO cambiar nada mas del className.

En el `<Image>` del logo (lineas ~56-62, `src='/brand/Isotipo.png'`):
- `width={80}` -> `width={160}`
- `height={80}` -> `height={160}`
- className `size-20 object-contain` -> `size-40 object-contain`

Esto duplica el tamano (80->160 / size-20->size-40) y, junto con `justify-items-center`, lo deja centrado.

Scope guard ESTRICTO: tocar SOLO el `<Image>` del logo y el className del `<CardHeader>` en este archivo. NO tocar el logo del sidebar, el texto de descripcion (`<p>`), los botones, ni `src/components/ui/card.tsx`. NO agregar dependencias.
  </action>
  <verify>
    <automated>cd nemea-front; npm run lint</automated>
    Tambien: `cd nemea-front; npm run build` debe completar sin errores.
    Verificacion visual: el usuario confirmara manualmente que el logo se ve 2x y centrado en /login.
  </verify>
  <done>
    - `page.tsx` tiene `width={160}`, `height={160}` y className `size-40 object-contain` en el `<Image>` del logo.
    - El `<CardHeader>` tiene `justify-items-center` en su className.
    - `npm run lint` y `npm run build` pasan sin errores nuevos.
    - Ningun otro archivo modificado.
  </done>
</task>

</tasks>

<commit_instructions>
El commit va en el sub-repo anidado `nemea-front` (repo git INDEPENDIENTE), en la branch existente `fix/login-logo`. NO commitear en el repo raiz `Nemea-web`.

Antes de commitear:
1. Verificar que estas parado en `nemea-front/` y en la branch `fix/login-logo`: `cd nemea-front; git branch --show-current` debe devolver `fix/login-logo`.
2. Stagear SOLO `src/app/(auth)/login/page.tsx` (no usar `git add -A`).
3. Mensaje sugerido: `fix(login): agrandar 2x y centrar el logo del leon`.
4. Incluir trailer `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`.

NO hacer push automatico. El push lo decide el usuario (ej. `/push f d`).
</commit_instructions>

<verification>
- `cd nemea-front; npm run lint` pasa.
- `cd nemea-front; npm run build` completa sin errores.
- Diff muestra cambios SOLO en `page.tsx`: `<Image>` (width/height/className) + className del `<CardHeader>`.
</verification>

<success_criteria>
- Logo del leon en /login renderiza a 160x160 (`size-40`), el doble del tamano previo.
- Logo centrado horizontalmente en el CardHeader.
- Sin cambios en otros assets ni en la primitiva `card.tsx`.
- Lint y build verdes.
</success_criteria>

<output>
Create `.planning/quick/260527-usx-agrandar-2x-y-centrar-el-logo-del-leon-e/260527-usx-SUMMARY.md` when done.
</output>
