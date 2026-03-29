---
status: complete
phase: 12-scenarios
source: [12-01-SUMMARY.md, 12-02-SUMMARY.md, 12-VERIFICATION.md, 12-03-SUMMARY.md]
started: "2026-03-29T05:30:00Z"
updated: "2026-03-29T07:00:00Z"
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
result: pass (retest after 12-03 gap closure)

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

### 9. Admin puede borrar escenarios públicos de otros usuarios
expected: Admin puede eliminar cualquier escenario público, incluso de otros usuarios. Necesario porque si un usuario se desactiva, sus escenarios públicos quedan huérfanos e inborrables.
result: pass (retest after 12-03 gap closure)

## Summary

total: 9
passed: 9
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

- truth: "Inactive products should appear in scenario editor with (inactivo) badge and reduced opacity"
  status: resolved
  reason: "Fixed in 12-03: added ?includeInactive=true to products fetch"
  severity: minor
  test: 4

- truth: "Admin can delete any public scenario (including from deactivated users) to prevent orphaned scenarios"
  status: resolved
  reason: "Fixed in 12-03: remove() accepts role param, admin bypasses owner filter, canDelete prop on ScenarioCard"
  severity: major
  test: 9
