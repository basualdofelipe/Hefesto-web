# Phase 12 — UI Review

**Audited:** 2026-03-29
**Baseline:** UI-SPEC.md (Phase 12 design contract, approved patterns)
**Screenshots:** Not captured (Playwright browser not installed — code-only audit)
**Scope:** Phase 12 Scenarios pages + broad audit of all (app)/ pages as requested

---

## Pillar Scores

| Pillar | Score | Key Finding |
|--------|-------|-------------|
| 1. Copywriting | 3/4 | Scenarios copy matches spec exactly; "Cancelar" appears as generic dismiss in several non-scenarios dialogs |
| 2. Visuals | 2/4 | Products page has 4-level cascade nesting (Type > Name > Finish > Row) that collapses into visual noise; scenarios table lists all SKUs ungrouped |
| 3. Color | 4/4 | Accent (terracotta primary) used precisely on brand marks, links, CTAs — no bleed into decorative elements |
| 4. Typography | 3/4 | Spec declares 2 weights (400, 600) but `font-medium` appears 59 times vs `font-bold` 3 times — 4 weights in practice |
| 5. Spacing | 4/4 | Scale is consistent; `space-y-1.5` is the only off-tick value and it appears only in form label-to-input gaps (intentional) |
| 6. Experience Design | 2/4 | Scenarios editor has no loading skeleton; ProductOverrideTable shows 26 individual SKUs with no product-name grouping; home page is placeholder-level |

**Overall: 18/24**

---

## Top 3 Priority Fixes

1. **ProductOverrideTable lists every SKU as a separate row** — With 26 SKUs, the user scrolls through "Billetera Hefesto Lisa Marrón", "Billetera Hefesto Lisa Negro", "Billetera Hefesto Cosida Marrón" etc. as independent rows. The user cannot quickly see "all Billeteras" at a glance. Fix: add a product-name group header row (non-collapsible is fine) so rows are visually chunked. This is a one-component change in `ProductOverrideTable.tsx` — insert a sticky group separator row whenever `product.name.name` changes within the sorted output.

2. **Products page 4-level cascade is cognitively expensive** — Level 1: Type collapsible (e.g. "Billeteras"), Level 2: Name collapsible (e.g. "Hefesto"), Level 3: Finish collapsible (e.g. "Lisa"), Level 4: SKU table rows. All four levels are open by default, so the page immediately shows hundreds of rows. Fix: change `ProductNameGroup` and `ProductFinishGroup` to initialize `isOpen = false` (collapsed by default) so the user starts with an overview and drills down intentionally. This is a 2-line change each.

3. **Home page (`/`) is a brand splash with zero utility** — The home page renders only "NEMEA" in large display font, a subtitle, and a theme toggle button. An authenticated user landing here gets no navigation value, no quick-link to their most-used pages, no at-a-glance summary. Fix: replace with a minimal dashboard widget — 3-4 cards linking to Productos, Calculadora, Insumos, and Gastos with their respective icons. Phase 13 (investor dashboard) will likely enhance this further, but even static quick-links would represent a large UX jump.

---

## Detailed Findings

### Pillar 1: Copywriting (3/4)

**Scenarios page — full contract compliance:**
- `escenarios/page.tsx:16` — "Escenarios" (matches spec)
- `escenarios/page.tsx:18` — "Simula cambios de precios y compara margenes sin afectar datos reales." (matches spec exactly)
- `ScenarioListClient.tsx:96` — "No tenes escenarios" (matches spec)
- `ScenarioListClient.tsx:101` — "Crear tu primer escenario" (matches spec)
- `ScenarioListClient.tsx:183` — "No eliminar" (matches spec)
- `ScenarioCard.tsx:99` — "Editar escenario" / "Ver escenario" (matches spec)
- `BulkOverrideDialog.tsx:72` — "Ajuste masivo de precios" (matches spec)
- All toast messages verified against spec contract.

**Minor gaps (non-scenarios pages):**
- Several dialogs across the app use "Cancelar" as a dismiss label (expenses: `DeleteExpenseDialog.tsx:78`, `ExpenseFormDialog.tsx:238`; products: `BatchPriceDialog.tsx:146`, `BomEditorDialog.tsx:197`, `BomGroupEditorDialog.tsx:332`; supplies: `SupplyFormDialog.tsx:329`). "Cancelar" is adequate Spanish UX but could be more specific (e.g., "No eliminar" pattern used in ScenarioListClient is stronger for destructive actions). This is a minor consistency gap, not a critical issue.
- `ScenarioEditorClient.tsx:97` — `toast('Overrides limpiados')` uses plain `toast()` instead of `toast.success()`. The spec says this is an informational toast; however the inconsistency with `toast.success('Escenario guardado y calculado')` on the same page creates an implicit urgency mismatch.
- `ScenarioEditorClient.tsx:116` — `toast('Override aplicado a ${count} productos')` — same issue, plain toast vs spec which says this should be informational (acceptable) but inconsistent with other success toasts on the page.

**Overall:** Scenarios copy is essentially perfect against spec. Broader app copy is in Spanish and contextually appropriate. Score deducted one point for the generic "Cancelar" pattern used broadly and the plain vs success toast inconsistency.

---

### Pillar 2: Visuals (2/4)

**Critical issue — Products page cascade depth:**

The product hierarchy renders 4 collapsible levels simultaneously:
```
Level 1 (ProductTypeGroup):  ▼ Billeteras  [12 productos]  Costo prom: $X
Level 2 (ProductNameGroup):    ▼ Hefesto  [6 productos]  ...
Level 3 (ProductFinishGroup):    ▼ Lisa  [3 productos]
Level 4 (Table rows inside):       SKU | Color | Talle | Costo | Precio | Margen
```

All four levels default to `isOpen = true` (`ProductTypeGroup.tsx:44`, `ProductNameGroup.tsx:56`, `ProductFinishGroup.tsx:55`). For a catalog with ~26 SKUs across 3 types, this renders 26 expanded table rows plus 3 type headers plus ~8 name headers plus ~12 finish headers simultaneously. The visual result is a very dense wall of data on first load.

The embedded table inside `ProductFinishGroup.tsx` renders at `ml-12` — this is the 4th level of left indentation. Combined with the small chevron icons shrinking from `size-4` (level 1) to `size-3.5` (level 2) to `size-3` (level 3), the visual hierarchy signal becomes very weak at lower levels.

**Notable issue — Scenarios editor shows ungrouped SKUs:**

`ProductOverrideTable.tsx` sorts products by type then name (`sortedProducts` in `ProductOverrideTable.tsx:96-105`), but renders every product as a flat table row with no visual group separator. With 26 rows, the user must mentally group "Billetera Hefesto Lisa Marrón", "Billetera Hefesto Lisa Negro", "Billetera Hefesto Cosida Marrón" — all adjacent rows — with no visual cue that they share a product name. The spec's `getProductDisplayName()` renders the full 4-part name in every row, creating a long repetitive product column.

**What works well:**
- ScenarioCard layout is clean. The name/date/gateway metadata row reads naturally. The Trash2 icon with Tooltip is properly sized (`h-4 w-4`) and the aria-label is correct.
- GatewayPlanSelector uses a horizontal flex-wrap layout that avoids the vertical stack of 5 selects that would have looked form-like.
- MarginSummary uses the correct 3-stat layout from the spec with clear labels.
- Blue highlight on overridden cells (`bg-blue-50 text-blue-700`) is immediately legible.
- TrendingUp/TrendingDown icons pair with color — both visual and iconographic signals for the difference column.
- ScenarioListClient renders the "Escenarios compartidos" section with a Separator, which is a good visual delineation.

**Home page:**
`app/(app)/page.tsx` renders only the brand name at `text-6xl`, a subtitle, and a theme toggle. This is a placeholder. An authenticated user landing on `/` has no navigation affordance beyond the sidebar. This is distinct from a "loading" issue — the page intentionally renders this content.

---

### Pillar 3: Color (4/4)

**Accent usage audit — 8 application-level uses of `text-primary`:**
1. `app/(app)/page.tsx:34` — "NEMEA" brand display title (correct: brand mark)
2. `app/(auth)/acceso-denegado/page.tsx:16` — "ACCESO DENEGADO" display title (correct: brand context)
3. `app/(auth)/login/page.tsx:54` — Login page brand title (correct)
4. `AppSidebar.tsx:100,103` — "NEMEA" / "N" in sidebar header (correct: brand mark)
5. `ProductFinishGroup.tsx:137` — SKU link in products table (correct: interactive link)
6. `ScenarioCard.tsx:54` — Scenario name link (correct: primary CTA per spec)
7. `SupplyExpandedRow.tsx:80` — Supply name link (correct: interactive link)

Zero accent misuse. The primary color is used exclusively on brand text and interactive links — consistent with the 60/30/10 spec. Hardcoded hex colors only appear in `login/page.tsx` for the Google OAuth button SVG icon (lines 16, 20, 24, 28: `#4285F4`, `#34A853`, `#FBBC05`, `#EA4335`). These are Google's brand colors embedded in the SVG path fill — this is correct and unavoidable.

Semantic colors in scenarios:
- `text-green-600 dark:text-green-400` — margin improvement (ProductOverrideTable.tsx:177,189)
- `text-red-600 dark:text-red-400` — margin decrease (ProductOverrideTable.tsx:179,191)
- `bg-blue-50 text-blue-700 dark:bg-blue-950/30 dark:text-blue-300` — override price highlight (ProductOverrideTable.tsx:163-166)

All three match the spec exactly and follow the DesglosePanel.tsx established pattern.

---

### Pillar 4: Typography (3/4)

**Sizes in use (from grep count):**

| Class | Usage count | Spec role |
|-------|-------------|-----------|
| `text-sm` | 147 | Body, table cells, labels |
| `text-xs` | 51 | Badges, sub-labels, table sub-info |
| `text-2xl` | 17 | Page titles, MarginSummary stats |
| `text-lg` | 10 | Section headings (empty state h2, product detail) |
| `text-base` | 7 | Card titles |
| `text-xl` | 2 | Sidebar brand, detail page |
| `text-4xl` | 1 | Login page brand title |
| `text-3xl` | 1 | Access denied page brand title |

The spec declares 6 typographic roles and none of these sizes are outside declared ranges. The `text-4xl` and `text-3xl` are in auth/brand pages where the large engraving font is intentional.

**Weights in use:**

| Class | Usage count | Issue |
|-------|-------------|-------|
| `font-medium` | 59 | Not in spec (spec declares only regular/semibold) |
| `font-semibold` | 31 | Correct |
| `font-normal` | 3 | Correct |
| `font-bold` | 3 | Not in spec |

The spec declares "Weights used: 2 (400 regular, 600 semibold)". In practice:
- `font-medium` (500) is used 59 times — primarily in `ProductExpandedRow.tsx` for data labels ("Producto", "SKU" field captions) and in product table cells
- `font-bold` appears 3 times — in `BomGroupEditorDialog.tsx`

The visual impact is minor (medium vs semibold difference is subtle), but it represents a drift from the declared 2-weight system. The spec's intent was clear binary contrast (regular text / semibold emphasis). Medium weight introduces a third tier that blurs the contrast hierarchy.

**Notable strength:** All financial values use `tabular-nums` — consistent across ProductOverrideTable, MarginSummary, and all calc result cells. The `font-mono tabular-nums` combination on the override price input is exactly as specified.

---

### Pillar 5: Spacing (4/4)

**Scale compliance:**

The dominant spacing values are:
- `gap-2` (8px) — 65 uses
- `space-y-2` (8px) — 42 uses
- `space-y-1` (4px) — 31 uses
- `gap-1` (4px) — 25 uses
- `space-y-4` / `gap-4` (16px) — 23 uses each
- `space-y-6` (24px) — 16 uses (page-level flow, matches spec)

All values are multiples of 4px (Tailwind's default 4px base unit). The spec declared xs=4, sm=8, md=16, lg=24 — this is precisely what's used.

**`space-y-1.5` (6px) appears 23 times** — primarily in dialog form fields as label-to-input gap (`CreateScenarioDialog.tsx:92,102,118`, `GatewayPlanSelector.tsx:139,158,174,193,212`, `BulkOverrideDialog.tsx:75,91`). The spec says "Exceptions: none" but `space-y-1.5` is a standard shadcn form layout pattern and produces tighter label-input coupling that reads naturally. This is acceptable in practice — it's not an arbitrary arbitrary value and is consistent with shadcn's own form examples.

**Arbitrary values** (bracket notation):
- Column width values (`w-[100px]`, `w-[110px]`, `w-[130px]`, `w-[90px]`) — used in ProductOverrideTable and ExpenseMonthGroup for table column widths. These are necessary and standard for table layouts.
- `grid-cols-[1fr_100px_60px_40px]` in BomEditorDialog and BomGroupEditorDialog — precise grid for BOM row layout. Appropriate.
- `min-w-[200px]`, `min-w-[160px]`, `min-w-[120px]` in selectors — appropriate min-width constraints for Select components.
- `min-h-[200px]`, `max-h-[250px]` in ProductBatchCreateDialog — scroll container height constraints. Appropriate.

None of these constitute spacing-scale violations; they are layout-constraint values, not padding/margin values.

---

### Pillar 6: Experience Design (2/4)

**Loading state coverage:**
- Scenarios editor (`ScenarioEditorClient.tsx`): The "Guardar y calcular" button shows a `Loader2` spinner and `disabled={saving}` during the save+calculate round-trip (`ScenarioEditorClient.tsx:221`). Good.
- Scenarios editor: **No skeleton on the margin columns during calculation.** The spec specified "margin columns show Skeleton" while calculating. Currently, the columns show `—` (em dash) until results arrive. This is functional but not the polished skeleton transition the spec declared.
- ScenarioListClient: **No skeleton for the initial list load.** The page is server-rendered (RSC), so data arrives before hydration. No client-side loading state is needed here.
- PriceHistoryDialog uses `Skeleton` correctly (both products and supplies versions, lines 78 and 77).
- ProductExpandedRow fetches BOM data on expand and shows "Cargando materiales..." text (`ProductExpandedRow.tsx:176`) — this is a text placeholder, not a skeleton. Minor downgrade.

**Error state coverage:**
- All toast.error() calls are present throughout the app. The scenarios feature covers: create error, delete error, toggle-public error, save+calc error.
- `ScenarioEditorClient.tsx` does not handle the 404 case (scenario not found) that the spec declared should redirect to /escenarios with a toast. The RSC page (`[id]/page.tsx`) likely handles this server-side via the apiFetch throwing, but there is no explicit redirect+toast in the code reviewed.

**Empty state coverage:**
- Scenarios list empty state: fully implemented with heading, body copy, and CTA (`ScenarioListClient.tsx:93-113`).
- Products table empty state: functional (`ProductTable.tsx:114-119`).
- Supplies table: has empty state per SupplyTypeGroup (assumed from SupplyTable pattern).
- Expenses table: has empty state per ExpenseTable.

**Disabled states:**
- "Guardar y calcular" disabled while `saving` (correct).
- "Aplicar ajuste" disabled when percentage is 0 or NaN (`BulkOverrideDialog.tsx:111-114`).
- All form submission buttons disabled while `isSubmitting`.

**Destructive action confirmation:**
- Scenario deletion uses AlertDialog with "Eliminar escenario" confirmation (correct).
- Supply deletion, expense deletion — both have AlertDialog patterns.

**UX gap — ProductOverrideTable initial state:**
Before the user clicks "Guardar y calcular", all margin columns show `—`. The user sees a 7-column table where 3 of the 7 columns are empty dashes. This is not an error state — it's intentional (margins are calculated on demand). However there is no callout or instruction to the user explaining that they need to click "Guardar y calcular" to see margins. A subtle helper text beneath the actions bar or a tooltip on the margin column headers would make this discoverability pattern explicit.

**UX gap — "Sin precio" state:**
The spec declares `"Sin precio"` text when a product has no currentPrice and no override. In `ProductOverrideTable.tsx:151-153`, `formatSellingPrice(product.currentPrice)` is called directly — if currentPrice is null, the output depends on `formatSellingPrice`'s null handling. The spec copy "Sin precio" is not explicitly rendered as a badge or distinct label in the table; it would appear as whatever `formatSellingPrice(null)` returns.

---

## Registry Safety

Registry audit: 0 third-party blocks. All shadcn components are from the official registry (Dialog, AlertDialog, Card, Table, Input, Select, Badge, Button, Switch, Skeleton, Separator, Tooltip). No flags.

---

## Files Audited

**Scenarios (Phase 12):**
- `nemea-front/src/app/(app)/escenarios/page.tsx`
- `nemea-front/src/app/(app)/escenarios/ScenarioListClient.tsx`
- `nemea-front/src/app/(app)/escenarios/[id]/page.tsx`
- `nemea-front/src/app/(app)/escenarios/[id]/ScenarioEditorClient.tsx`
- `nemea-front/src/components/scenarios/ScenarioCard.tsx`
- `nemea-front/src/components/scenarios/CreateScenarioDialog.tsx`
- `nemea-front/src/components/scenarios/ProductOverrideTable.tsx`
- `nemea-front/src/components/scenarios/BulkOverrideDialog.tsx`
- `nemea-front/src/components/scenarios/GatewayPlanSelector.tsx`
- `nemea-front/src/components/scenarios/MarginSummary.tsx`

**Products page (broad audit):**
- `nemea-front/src/app/(app)/productos/page.tsx`
- `nemea-front/src/components/products/ProductTable.tsx`
- `nemea-front/src/components/products/ProductTypeGroup.tsx`
- `nemea-front/src/components/products/ProductNameGroup.tsx`
- `nemea-front/src/components/products/ProductFinishGroup.tsx`
- `nemea-front/src/components/products/ProductExpandedRow.tsx`

**Other pages (broad audit):**
- `nemea-front/src/app/(app)/page.tsx`
- `nemea-front/src/app/(app)/insumos/page.tsx`
- `nemea-front/src/app/(app)/finanzas/gastos/page.tsx`
- `nemea-front/src/components/layout/AppSidebar.tsx`
- `nemea-front/src/app/globals.css`
- `nemea-front/src/components/supplies/SupplyTable.tsx` (partial)
- `nemea-front/src/app/(app)/calculadora/CalculadoraClient.tsx` (partial)
