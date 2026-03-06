---
phase: quick
plan: 1
type: execute
wave: 1
depends_on: []
files_modified:
  - nemea-front/src/components/products/ProductTypeGroup.tsx
  - nemea-back/src/catalogs/catalogs.service.ts
autonomous: true
requirements: []
must_haves:
  truths:
    - "ProductTypeGroup renders without hydration errors or button nesting warnings"
    - "Creating catalog items in all 6 dimensions succeeds (no 500 error)"
    - "New product dimension items get auto-assigned sequential skuCode"
  artifacts:
    - path: "nemea-front/src/components/products/ProductTypeGroup.tsx"
      provides: "Fixed collapsible trigger without nested buttons"
    - path: "nemea-back/src/catalogs/catalogs.service.ts"
      provides: "Auto-assign skuCode on create for product dimensions"
  key_links:
    - from: "catalogs.service.ts create()"
      to: "product dimension entities"
      via: "MAX(sku_code) + 1 query before insert"
      pattern: "createQueryBuilder.*MAX.*sku_code"
---

<objective>
Fix two bugs discovered during phase 5 verification:

1. **Hydration error**: CollapsibleTrigger renders as `<button>` and contains Button components (Editar BOM grupal, Precio grupal), violating HTML nesting rules and causing React hydration mismatch.

2. **Catalog creation broken**: Creating items in product dimensions (product-types, product-names, product-finishes, product-colors, product-sizes) fails with 500 because the generic `create()` method doesn't auto-assign the NOT NULL `skuCode` column added in plan 05-01. Supply-types has no skuCode and works fine.

Purpose: Restore basic CRUD functionality and eliminate console errors.
Output: Two patched files, both bugs resolved.
</objective>

<execution_context>
@./.claude/get-shit-done/workflows/execute-plan.md
@./.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@nemea-front/src/components/products/ProductTypeGroup.tsx
@nemea-back/src/catalogs/catalogs.service.ts
@nemea-back/src/catalogs/entities/product-color.entity.ts
@nemea-back/src/catalogs/dto/create-catalog-item.dto.ts
</context>

<tasks>

<task type="auto">
  <name>Task 1: Fix button nesting in ProductTypeGroup CollapsibleTrigger</name>
  <files>nemea-front/src/components/products/ProductTypeGroup.tsx</files>
  <action>
The CollapsibleTrigger (line 69) renders as a `<button>` and contains two Button components inside it (lines 84-101). This is invalid HTML and causes hydration errors.

Fix approach: Use `asChild` on CollapsibleTrigger so it delegates rendering to a child `<div>` instead of rendering its own `<button>`. This keeps the Radix open/close behavior while avoiding button nesting.

Replace the current CollapsibleTrigger block (lines 69-104) with:

```tsx
<CollapsibleTrigger asChild>
  <div
    role="button"
    tabIndex={0}
    className="hover:bg-muted/50 flex w-full items-center gap-2 rounded-md px-3 py-2 text-left transition-colors cursor-pointer"
    onKeyDown={(e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        setIsOpen(!isOpen);
      }
    }}
  >
    <ChevronDown
      className={`size-4 shrink-0 transition-transform duration-200 ${
        isOpen ? '' : '-rotate-90'
      }`}
    />
    <span className="font-medium">{typeName}</span>
    <Badge variant="secondary" className="ml-1">
      {products.length} {products.length === 1 ? 'producto' : 'productos'}
    </Badge>
    {isAdmin && (
      <div
        className="ml-auto flex items-center gap-1"
        onClick={(e) => e.stopPropagation()}
      >
        <Button
          size="sm"
          variant="ghost"
          className="h-7 text-xs"
          onClick={() => setShowGroupBomEditor(true)}
        >
          <ClipboardList className="mr-1 size-3" />
          Editar BOM grupal
        </Button>
        <Button
          size="sm"
          variant="ghost"
          className="h-7 text-xs"
          onClick={() => setShowBatchPrice(true)}
        >
          <DollarSign className="mr-1 size-3" />
          Precio grupal
        </Button>
      </div>
    )}
  </div>
</CollapsibleTrigger>
```

The `asChild` prop makes Radix merge its props onto the child element instead of rendering its own button. The div gets role="button", tabIndex=0, and keyboard handling for a11y.
  </action>
  <verify>
    <automated>cd nemea-front && npx next build 2>&1 | tail -20</automated>
  </verify>
  <done>CollapsibleTrigger no longer renders a button element. No nested button HTML. Build succeeds without hydration-related warnings.</done>
</task>

<task type="auto">
  <name>Task 2: Auto-assign skuCode in CatalogsService.create() for product dimensions</name>
  <files>nemea-back/src/catalogs/catalogs.service.ts</files>
  <action>
The `create()` method (line 98) does `repo.create(dto)` which only sets `name`. Product dimension entities (product-types, product-names, product-finishes, product-colors, product-sizes) have a NOT NULL `skuCode` column, so inserts fail with a constraint violation. Supply-types has NO skuCode and must not be affected.

Fix: In the `create()` method, detect if the dimension needs a skuCode (all dimensions except 'supply-types') and auto-assign the next sequential value.

Define a set of dimensions that need skuCode at the top of the class:

```typescript
private readonly DIMENSIONS_WITH_SKU: ReadonlySet<string> = new Set([
  'product-types',
  'product-names',
  'product-finishes',
  'product-colors',
  'product-sizes',
]);
```

Modify the `create()` method to auto-assign skuCode when:
1. The dimension is in DIMENSIONS_WITH_SKU, AND
2. `dto.skuCode` is not already provided (undefined/null)

To get the next skuCode, query the repository:
```typescript
if (this.DIMENSIONS_WITH_SKU.has(dimension) && dto.skuCode == null) {
  const result = await repo
    .createQueryBuilder('item')
    .select('COALESCE(MAX(item.skuCode), 0)', 'maxCode')
    .getRawOne();
  dto.skuCode = (parseInt(result.maxCode, 10) || 0) + 1;
}
```

This uses MAX + 1 pattern. The COALESCE handles empty tables (returns 0, so first item gets skuCode=1).

Make sure this is placed BEFORE the `repo.create(dto)` call inside the try block. The dto type already has `skuCode?: number` as optional, so mutating it is type-safe.
  </action>
  <verify>
    <automated>cd nemea-back && npx nest build 2>&1 | tail -10</automated>
  </verify>
  <done>Creating catalog items in any of the 6 dimensions succeeds. Product dimensions auto-assign sequential skuCode. Supply-types (no skuCode column) unaffected.</done>
</task>

</tasks>

<verification>
1. `cd nemea-front && npx next build` completes without errors
2. `cd nemea-back && npx nest build` compiles without errors
3. Manual: Open products page, confirm no console errors about button nesting or hydration
4. Manual: Create a new item in Catalogos > Colores (or any product dimension), confirm it succeeds and gets a skuCode
</verification>

<success_criteria>
- No HTML nesting violations in ProductTypeGroup (no button-in-button)
- All 6 catalog dimensions support item creation without errors
- Product dimension items auto-receive sequential skuCode values
- Both projects build cleanly
</success_criteria>

<output>
After completion, create `.planning/quick/1-fix-button-nesting-hydration-error-and-c/1-SUMMARY.md`
</output>
