---
status: complete
phase: 12-scenarios
source: [12-01-SUMMARY.md, 12-02-SUMMARY.md, 12-VERIFICATION.md]
started: "2026-03-29T05:30:00Z"
updated: "2026-03-29T06:00:00Z"
---

## Current Test

[testing complete]

## Tests

### 1. Cold Start Smoke Test
expected: Backend arranca sin errores. GET /api/scenarios devuelve array vacío.
result: pass

### 2. Sidebar link visible para todos
expected: En el sidebar aparece "Escenarios" bajo la sección Herramientas, visible tanto para admin como para user.
result: pass

### 3. Crear escenario y ver en lista
expected: Click en "Crear escenario" abre dialog. Completar nombre (ej "Aumento Enero 2027"), seleccionar gateway/plan. Se crea y aparece como card en /escenarios.
result: pass

### 4. Editor con override de precios
expected: Entrar al escenario. Tabla muestra todos los productos con costo y precio real. Se puede editar el precio override de un producto (input azul cuando tiene valor). Los inactivos aparecen con badge "(inactivo)" y opacity reducida.
result: issue
reported: "los inactivos no aparecen, no creo que sea un problema que no aparezcan, pero hay que chequear la lógica porque no hace lo que pretendés"
severity: minor

### 5. Guardar y calcular — comparación de márgenes
expected: Click "Guardar y calcular". Spinner, luego columnas de margen se llenan. Verde = margen mejoró, rojo = empeoró. MarginSummary card muestra resumen.
result: pass

### 6. Ajuste masivo por tipo
expected: Click "Ajuste masivo". Seleccionar tipo de producto + porcentaje (ej +10%). Preview muestra cuántos productos afecta. Aplicar actualiza los overrides en la tabla.
result: pass

### 7. Toggle público/privado
expected: En la card del escenario o en el editor, toggle de público/privado funciona. Los escenarios públicos son visibles para otros usuarios.
result: pass

### 8. Eliminar escenario
expected: Click en eliminar pide confirmación. Confirmar borra el escenario y desaparece de la lista.
result: pass

## Summary

total: 8
passed: 7
issues: 1
pending: 0
skipped: 0
blocked: 0

## Gaps

- truth: "Inactive products should appear in scenario editor with (inactivo) badge and reduced opacity"
  status: failed
  reason: "User reported: inactive products don't appear at all in the scenario editor table"
  severity: minor
  test: 4
  artifacts:
    - nemea-front/src/app/(app)/escenarios/[id]/ScenarioEditorClient.tsx
    - nemea-back/src/scenarios/scenarios.service.ts
  missing: []
