# Phase 5: Products and BOM - Research

**Researched:** 2026-03-05
**Domain:** Product CRUD, Bill of Materials (BOM) versioning, SKU generation, selling price history
**Confidence:** HIGH

## Summary

Phase 5 builds Product CRUD with 5 catalog dimension FKs, auto-generated SKU from dimension sku_codes, BOM (supplies_per_product_history) with version tracking via is_active flag, and selling price history (product_price_history). This phase follows established patterns from Phase 4 (Supplies) very closely -- grouped table with expand inline, modals for CRUD, inline price addition, and price history dialog.

The key complexity is the BOM versioning system (old BOM records marked is_active=false in a transaction when a new BOM is saved), batch product creation (cartesian product of colors x sizes), and the BOM group editing feature. The backend needs a new ProductsModule with entities for Product, SuppliesPerProductHistory, and ProductPriceHistory. The frontend reuses the SupplyTypeGroup/SupplyExpandedRow pattern but adds a multi-step batch creation modal and a BOM editor modal with supply Combobox.

**Primary recommendation:** Follow the exact Supplies module pattern (entity, service, controller, DTOs, module) for Products, reuse frontend components (AddPriceInline, PriceHistoryDialog patterns), and implement BOM versioning as a transactional swap (mark old active=false, insert new active=true).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- SKU format: `{type.sku_code}.{name.sku_code}.{finish.sku_code}.{color.sku_code}.{size.sku_code}` (e.g., `1.2.1.3.0`)
- Each catalog entity gets a new `sku_code` field (smallint, unique) assigned manually by admin
- Migration adds `sku_code` column to all 5 product dimension tables + product_size
- SKU auto-generated on product creation by concatenating the sku_codes of selected dimensions
- "Talle Unico" seeded as default size with sku_code 0
- Dimensions editable after creation -- SKU regenerates automatically on edit
- `sku_code` stored as VARCHAR in product table, unique constraint
- Product list: grouped table by product type with collapsible sections
- Table columns: SKU, Nombre, Terminacion, Color, Talle, Precio Venta
- Product expand inline shows: BOM table, precio de venta actual, action buttons
- Batch product creation: Step 1 (tipo+nombre+terminacion) -> Step 2 (checkbox colores) -> Step 3 (checkbox talles) -> Preview -> Create
- BOM editor: modal with editable table, select supply via Combobox grouped by type, quantity input, unit auto-filled from supply.unitType
- Same supply can appear multiple times in BOM (no unique constraint on product_id + supply_id)
- BOM save creates new version: old records is_active=false, new records is_active=true (transaction)
- BOM group editing: button in group header, checkboxes of products, warning for different BOMs, standard BOM editor below
- Supply deactivation guard: block deactivation if supply is in any active BOM
- Selling price: inline add, append-only history, batch price update from group header
- Currency field in DB (VARCHAR default 'ARS') but NOT shown in UI
- Single page at `/productos`, everything via expand inline + modals
- "Productos" added to sidebar under "Datos base" group
- Seed real products from Costos Nemea.xlsx with BOM
- Seed sku_code values for existing catalog items
- Seed as TypeORM migration (idempotent, ON CONFLICT DO NOTHING)

### Claude's Discretion
- Exact styling of group headers, expand rows, and modals
- Loading and error states
- Toast messages and duration
- Empty state design for products list and BOM
- Search debounce timing
- Combobox design for supply selection in BOM editor
- How to detect "majority BOM" for group edit warning
- Collapsible sections default state

### Deferred Ideas (OUT OF SCOPE)
- BOM version history UI -- records stored in DB (is_active=false) but no UI to view old versions yet
- Margin calculation (precio venta - costo) -- Phase 6 when cost is calculated
- Price per channel (Tiendanube, B2B, direct) -- v2
- Visual price trend indicators (arrows, percentages) -- v2 polish
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PROD-01 | Admin can create a product by selecting type, name, finish, color, and size | Product entity with 5 FK dimensions, CreateProductDto with batch support, batch creation endpoint |
| PROD-02 | Each product has an auto-generated unique SKU (sku_code) | Migration adds sku_code to 5 catalog tables, SKU generated as dot-separated concatenation, unique constraint on product.sku_code |
| PROD-03 | Admin can edit product attributes | UpdateProductDto, PUT endpoint, SKU regeneration on dimension change |
| PROD-04 | Admin can deactivate/reactivate products (soft delete) | is_active column, PATCH toggle-status endpoint, partial unique index pattern from supplies |
| PROD-05 | Admin can define the material composition (BOM) of a product | SuppliesPerProductHistory entity, BOM editor modal, PUT /products/:id/bom endpoint |
| PROD-06 | When BOM changes, old composition is preserved as history (is_active flag) | Transactional BOM swap: mark old is_active=false, insert new is_active=true |
| PROD-07 | Admin can add a selling price to a product (historical record) | ProductPriceHistory entity, POST /products/:id/prices endpoint, reuses AddPriceInline pattern |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Backend framework | Already in project |
| TypeORM | 0.3.x | ORM + migrations | Already in project |
| class-validator | 0.14.x | DTO validation | Already in project |
| class-transformer | 0.5.x | DTO transformation | Already in project |
| Next.js | 16.x | Frontend framework | Already in project |
| react-hook-form | 7.x | Form management | Already in project |
| zod | 3.x | Schema validation | Already in project |
| @hookform/resolvers | 3.x | RHF + zod bridge | Already in project |
| Shadcn/ui | latest | UI components | Already in project |
| sonner | latest | Toast notifications | Already in project |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| cmdk | 1.x | Combobox component | BOM editor supply selection (Shadcn Command component) |
| lucide-react | latest | Icons | Already in project |

### No New Dependencies Needed
The entire phase can be built with existing project dependencies. The Combobox for BOM editor uses Shadcn's Command component which wraps cmdk (already installed with Shadcn).

## Architecture Patterns

### Backend Structure (mirror Supplies module)
```
nemea-back/src/
├── products/
│   ├── entities/
│   │   ├── product.entity.ts
│   │   ├── supplies-per-product-history.entity.ts
│   │   └── product-price-history.entity.ts
│   ├── dto/
│   │   ├── create-product.dto.ts
│   │   ├── create-batch-products.dto.ts
│   │   ├── update-product.dto.ts
│   │   ├── update-bom.dto.ts
│   │   ├── create-product-price.dto.ts
│   │   └── batch-product-price.dto.ts
│   ├── products.controller.ts
│   ├── products.service.ts
│   └── products.module.ts
├── database/migrations/
│   ├── 1772500000000-AddSkuCodeToCatalogs.ts
│   ├── 1772500100000-CreateProductsAndBom.ts
│   └── 1772500200000-SeedProductsAndBom.ts
```

### Frontend Structure (mirror Supplies page)
```
nemea-front/src/
├── app/(app)/productos/
│   └── page.tsx                    # Server component: fetch products + catalogs + supplies
├── components/products/
│   ├── types.ts                    # Product, BomItem, PriceRecord interfaces
│   ├── ProductTable.tsx            # Main table with search, filters, group header actions
│   ├── ProductTypeGroup.tsx        # Collapsible group by product type (mirrors SupplyTypeGroup)
│   ├── ProductExpandedRow.tsx      # Expand inline with BOM, price, actions
│   ├── ProductBatchCreateDialog.tsx # Multi-step batch creation modal
│   ├── ProductEditDialog.tsx       # Single product edit modal
│   ├── BomEditorDialog.tsx         # BOM editor modal with supply Combobox
│   ├── BomGroupEditorDialog.tsx    # BOM group editor (header button)
│   ├── AddSellingPriceInline.tsx   # Reuses AddPriceInline pattern
│   ├── PriceHistoryDialog.tsx      # Reuses supplies PriceHistoryDialog pattern
│   └── BatchPriceDialog.tsx        # Batch price update from group header
```

### Pattern 1: Product Entity with 5 FK Dimensions
**What:** Product has ManyToOne to ProductType, ProductName, ProductFinish, ProductColor, ProductSize
**When to use:** Product creation and SKU generation
**Example:**
```typescript
// Source: existing Supply entity pattern + CONTEXT.md decisions
@Entity('products')
export class Product extends BaseEntity {
  @Column({ name: 'sku_code', type: 'varchar', length: 50, unique: true })
  skuCode!: string;

  @ManyToOne(() => ProductType, { eager: true, nullable: false })
  @JoinColumn({ name: 'product_type_id' })
  type!: ProductType;

  @ManyToOne(() => ProductName, { eager: true, nullable: false })
  @JoinColumn({ name: 'product_name_id' })
  name!: ProductName;

  @ManyToOne(() => ProductFinish, { eager: true, nullable: false })
  @JoinColumn({ name: 'product_finish_id' })
  finish!: ProductFinish;

  @ManyToOne(() => ProductColor, { eager: true, nullable: false })
  @JoinColumn({ name: 'product_color_id' })
  color!: ProductColor;

  @ManyToOne(() => ProductSize, { eager: true, nullable: false })
  @JoinColumn({ name: 'product_size_id' })
  size!: ProductSize;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}
```

### Pattern 2: BOM Versioning with is_active Flag
**What:** When BOM changes, all old active records are marked is_active=false and new records inserted as is_active=true, in a single transaction
**When to use:** PUT /products/:id/bom endpoint
**Example:**
```typescript
// Source: CONTEXT.md decision + existing transaction pattern from SuppliesService
async updateBom(productId: string, dto: UpdateBomDto): Promise<SuppliesPerProductHistory[]> {
  return this.entityManager.transaction(async (manager) => {
    // Mark all current active BOM entries as inactive
    await manager.update(
      SuppliesPerProductHistory,
      { product: { id: productId }, isActive: true },
      { isActive: false },
    );

    // Insert new BOM entries
    const newEntries = dto.items.map((item) =>
      manager.create(SuppliesPerProductHistory, {
        product: { id: productId } as Product,
        supply: { id: item.supplyId } as Supply,
        quantity: String(item.quantity),
        isActive: true,
      }),
    );

    return manager.save(SuppliesPerProductHistory, newEntries);
  });
}
```

### Pattern 3: SKU Generation
**What:** SKU auto-generated by concatenating sku_code values from the 5 selected dimensions
**When to use:** Product creation and editing
**Example:**
```typescript
// Source: CONTEXT.md decision
private generateSkuCode(
  type: ProductType,
  name: ProductName,
  finish: ProductFinish,
  color: ProductColor,
  size: ProductSize,
): string {
  return `${type.skuCode}.${name.skuCode}.${finish.skuCode}.${color.skuCode}.${size.skuCode}`;
}
```

### Pattern 4: Batch Product Creation
**What:** Create N products from cartesian product of selected colors x sizes, with fixed type+name+finish
**When to use:** POST /products/batch endpoint
**Example:**
```typescript
// Source: CONTEXT.md decision
async createBatch(dto: CreateBatchProductsDto): Promise<Product[]> {
  return this.entityManager.transaction(async (manager) => {
    const type = await this.findCatalogItem(manager, ProductType, dto.typeId);
    const name = await this.findCatalogItem(manager, ProductName, dto.nameId);
    const finish = await this.findCatalogItem(manager, ProductFinish, dto.finishId);
    const colors = await this.findCatalogItems(manager, ProductColor, dto.colorIds);
    const sizes = await this.findCatalogItems(manager, ProductSize, dto.sizeIds);

    const products: Product[] = [];
    for (const color of colors) {
      for (const size of sizes) {
        const skuCode = this.generateSkuCode(type, name, finish, color, size);
        const product = manager.create(Product, {
          skuCode,
          type, name, finish, color, size,
        });
        products.push(product);
      }
    }

    return manager.save(Product, products);
  });
}
```

### Pattern 5: Supply Deactivation Guard
**What:** Before deactivating a supply, check if it's used in any active BOM
**When to use:** Enhancement to existing SuppliesService.toggleStatus
**Example:**
```typescript
// Source: CONTEXT.md decision
// In SuppliesService.toggleStatus, before setting isActive=false:
if (supply.isActive) {
  const bomUsage = await this.bomRepo.count({
    where: { supply: { id }, isActive: true },
  });
  if (bomUsage > 0) {
    throw new ConflictException(
      `Este insumo esta en uso por ${bomUsage} productos`,
    );
  }
}
```

### Anti-Patterns to Avoid
- **Eager-loading all BOM items on product list:** Only load BOM when expanding a product row. The product list endpoint should NOT include BOM data -- fetch BOM separately via GET /products/:id/bom.
- **Using JSONB for BOM:** The original design considered JSONB. The current design uses a normalized table (supplies_per_product_history) which enables the supply deactivation guard and future cost calculation queries.
- **Regenerating SKU in frontend:** SKU generation MUST happen in the backend to guarantee uniqueness. Frontend shows a preview but backend is authoritative.
- **Deleting BOM records:** Never DELETE BOM records. Always use the is_active flag swap pattern.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Combobox with search | Custom dropdown with filtering | Shadcn Command/Combobox pattern | Handles keyboard navigation, search, groups |
| Multi-step modal | Custom step state machine | Numbered steps with conditional rendering | Shadcn Dialog + internal step state is simpler |
| Batch transaction | Manual try/catch with rollback | TypeORM EntityManager.transaction() | Handles rollback automatically on error |
| Decimal precision | JavaScript number arithmetic | decimal(12,2) in DB + string transport | Avoids floating point errors (same as supply prices) |
| Unique constraint check | Manual SELECT before INSERT | Partial unique index + catch 23505 | Race-condition safe (same as supplies pattern) |

**Key insight:** Nearly every backend pattern exists in the Supplies module. The BOM versioning is the only truly new pattern, and it's a straightforward transactional swap.

## Common Pitfalls

### Pitfall 1: SKU Uniqueness Race Condition in Batch Creation
**What goes wrong:** Two concurrent batch creations could try to create the same SKU (same type+name+finish+color+size combination)
**Why it happens:** Without proper constraints, the unique check happens before the insert
**How to avoid:** Unique constraint on product.sku_code column + catch PostgreSQL error code 23505 (unique violation). The constraint handles the race condition at DB level.
**Warning signs:** ConflictException not being thrown on duplicate SKU

### Pitfall 2: BOM Version Swap Not Atomic
**What goes wrong:** If the UPDATE (mark old inactive) succeeds but the INSERT (new entries) fails, the product ends up with no active BOM
**Why it happens:** Not wrapping both operations in a transaction
**How to avoid:** Always use EntityManager.transaction() for BOM updates. Both operations must succeed or both rollback.
**Warning signs:** Products with zero active BOM entries after a failed save

### Pitfall 3: sku_code Column on Catalog Tables Must Handle Existing Data
**What goes wrong:** Adding a NOT NULL sku_code column to tables that already have seeded data causes migration failure
**Why it happens:** Existing rows have no value for the new column
**How to avoid:** Migration strategy: (1) ADD COLUMN sku_code SMALLINT NULL, (2) UPDATE existing rows with seed sku_code values, (3) ALTER COLUMN SET NOT NULL + ADD UNIQUE CONSTRAINT. Do this in a single migration.
**Warning signs:** Migration fails on `NOT NULL` constraint

### Pitfall 4: Supply unitType Not Available in BOM Editor
**What goes wrong:** BOM editor needs to show the unit type of each supply, but the supply list endpoint may not include it
**Why it happens:** The supplies endpoint already returns unitType, but the frontend needs to pass the full supply list to the BOM editor
**How to avoid:** The productos page server component must fetch supplies (with unitType) in addition to products. Pass to BOM editor for auto-fill.
**Warning signs:** Unit column shows "undefined" in BOM editor

### Pitfall 5: Product Display Name Construction
**What goes wrong:** The "name" column in the product list shows only the ProductName dimension, but the user expects the full product name
**Why it happens:** The product entity has separate type, name, finish, color, size -- the display name is a concatenation
**How to avoid:** Create a helper function `getProductDisplayName(product)` that returns e.g. "Billetera Hefesto Lisa Marron Unico". Use consistently in list, expanded row, and dialogs.
**Warning signs:** Inconsistent product naming across UI

### Pitfall 6: Partial Unique Index for Products
**What goes wrong:** Deactivated products block creation of new products with the same dimension combination
**Why it happens:** Unique constraint on sku_code applies to all rows including inactive
**How to avoid:** Use a partial unique index on sku_code WHERE is_active = true (same pattern as supplies). This allows a product to be deactivated and a new one with the same SKU to be created.
**Warning signs:** "Product already exists" when creating a product whose SKU was previously deactivated

### Pitfall 7: BOM Group Edit Majority Detection
**What goes wrong:** "Majority BOM" detection is ambiguous when products have different BOMs
**Why it happens:** Comparing BOMs requires comparing sets of {supplyId, quantity} tuples
**How to avoid:** Serialize each product's active BOM as a sorted JSON string of [{supplyId, quantity}], group by that string, pick the group with the most products as "majority". Products not in the majority group get a warning indicator.
**Warning signs:** Warning indicators showing incorrectly or not at all

## Code Examples

### Entity: SuppliesPerProductHistory (BOM)
```typescript
// Source: db-model-draft.md + existing SupplyPriceHistory pattern
@Entity('supplies_per_product_history')
export class SuppliesPerProductHistory extends BaseEntity {
  @ManyToOne(() => Product, { nullable: false, onDelete: 'CASCADE' })
  @JoinColumn({ name: 'product_id' })
  product!: Product;

  @ManyToOne(() => Supply, { nullable: false })
  @JoinColumn({ name: 'supply_id' })
  supply!: Supply;

  @Column({ type: 'decimal', precision: 10, scale: 2, nullable: false })
  quantity!: string; // decimal stored as string (TypeORM pattern)

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}
```

Note: `unit_type` is NOT stored in the BOM table. It comes from `supply.unitType` and is displayed read-only in the BOM editor. This avoids data duplication and stays consistent with the supply's unit.

### Entity: ProductPriceHistory
```typescript
// Source: existing SupplyPriceHistory pattern
@Entity('product_price_history')
export class ProductPriceHistory extends BaseEntity {
  @ManyToOne(() => Product, { nullable: false, onDelete: 'CASCADE' })
  @JoinColumn({ name: 'product_id' })
  product!: Product;

  @Column({ type: 'decimal', precision: 12, scale: 2, nullable: false })
  price!: string;

  @Column({ type: 'varchar', length: 10, default: 'ARS' })
  currency!: string;
}
```

### Migration: Add sku_code to Catalog Tables
```typescript
// Source: CONTEXT.md decision + existing migration patterns
public async up(queryRunner: QueryRunner): Promise<void> {
  // Add sku_code column (nullable initially)
  for (const table of ['product_types', 'product_names', 'product_finishes', 'product_colors', 'product_sizes']) {
    await queryRunner.query(
      `ALTER TABLE "${table}" ADD COLUMN "sku_code" SMALLINT NULL`,
    );
  }

  // Seed sku_code values for existing catalog items
  // (values based on user's existing system from Costos Nemea.xlsx)
  await queryRunner.query(`UPDATE "product_types" SET "sku_code" = 1 WHERE "name" = 'Billetera'`);
  // ... more updates ...

  // Make NOT NULL + add unique constraint
  for (const table of ['product_types', 'product_names', 'product_finishes', 'product_colors', 'product_sizes']) {
    await queryRunner.query(
      `ALTER TABLE "${table}" ALTER COLUMN "sku_code" SET NOT NULL`,
    );
    await queryRunner.query(
      `ALTER TABLE "${table}" ADD CONSTRAINT "UQ_${table}_sku_code" UNIQUE ("sku_code")`,
    );
  }
}
```

### API Endpoints
```
GET    /products?includeInactive=false    # List with current selling price
GET    /products/:id                       # Single product detail
POST   /products                           # Create single product
POST   /products/batch                     # Batch create (colors x sizes)
PUT    /products/:id                       # Update product (regenerates SKU)
PATCH  /products/:id/toggle-status         # Soft delete toggle

GET    /products/:id/bom                   # Get active BOM
PUT    /products/:id/bom                   # Replace BOM (version swap)
PUT    /products/batch-bom                 # Group BOM update

GET    /products/:id/prices                # Price history
POST   /products/:id/prices               # Add selling price
POST   /products/batch-prices              # Batch price update
```

### Frontend: Server Component Data Fetching
```typescript
// Source: existing insumos/page.tsx pattern
export default async function ProductosPage(): Promise<ReactElement> {
  const session = await auth();
  const isAdmin = session?.user?.role === 'admin';

  const [productsRes, typesRes, namesRes, finishesRes, colorsRes, sizesRes, suppliesRes] = await Promise.all([
    apiFetch<{ data: Product[] }>('/api/products?includeInactive=true'),
    apiFetch<{ data: CatalogItem[] }>('/api/catalogs/product-types'),
    apiFetch<{ data: CatalogItem[] }>('/api/catalogs/product-names'),
    apiFetch<{ data: CatalogItem[] }>('/api/catalogs/product-finishes'),
    apiFetch<{ data: CatalogItem[] }>('/api/catalogs/product-colors'),
    apiFetch<{ data: CatalogItem[] }>('/api/catalogs/product-sizes'),
    apiFetch<{ data: Supply[] }>('/api/supplies'),
  ]);
  // ... render ProductTable with all data
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| JSONB for BOM | Normalized table with is_active versioning | Design phase (db-model-draft) | Enables FK constraints, deactivation guard, cost queries |
| product.date column | BaseEntity.createdAt | Phase 3 (UUID migration) | Consistent timestamps across all entities |
| SERIAL PKs | UUID PKs | Phase 3 (03-01) | All entities use BaseEntity with UUID |

## Open Questions

1. **Costos Nemea.xlsx Product Seed Data**
   - What we know: The seed should include real products from the Excel file with their BOM
   - What's unclear: Exact sku_code assignments for existing catalog items need to be extracted from the Excel
   - Recommendation: Read the Excel during plan execution to extract exact seed values. The executor should reference `Costos Nemea.xlsx` for real product data.

2. **Catalog Entity DTO Updates for sku_code**
   - What we know: CatalogsModule uses a generic dimension-to-repository map for all 5 dimension entities
   - What's unclear: Whether the generic CRUD approach can accommodate the new sku_code field or needs per-entity DTOs
   - Recommendation: Extend the generic CreateCatalogItemDto to include optional sku_code field. The generic approach should still work since sku_code is the same field across all 5 tables.

3. **BOM Editor: Lazy Loading Supplies**
   - What we know: The supplies list is needed for the BOM editor Combobox
   - What's unclear: Whether to fetch all supplies on page load or lazy-load when BOM editor opens
   - Recommendation: Fetch supplies on page load (same pattern as insumos page fetches suppliers). The supply list is small enough (likely < 100 items) that eager loading is fine.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (backend e2e) + Jest (frontend unit) |
| Config file | `nemea-back/test/jest-e2e.json` (backend), `nemea-front/jest.config.ts` or similar (frontend) |
| Quick run command | `cd nemea-back && npx jest --config test/jest-e2e.json --testPathPattern products` |
| Full suite command | `cd nemea-back && npx jest --config test/jest-e2e.json` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PROD-01 | Create product with 5 dimensions | e2e | `npx jest --config test/jest-e2e.json --testPathPattern products` | No - Wave 0 |
| PROD-02 | Auto-generated unique SKU | e2e | Same as above | No - Wave 0 |
| PROD-03 | Edit product, SKU regenerates | e2e | Same as above | No - Wave 0 |
| PROD-04 | Deactivate/reactivate product | e2e | Same as above | No - Wave 0 |
| PROD-05 | Define BOM for a product | e2e | Same as above | No - Wave 0 |
| PROD-06 | BOM change preserves old with is_active=false | e2e | Same as above | No - Wave 0 |
| PROD-07 | Add selling price, history preserved | e2e | Same as above | No - Wave 0 |

### Sampling Rate
- **Per task commit:** Manual testing via Swagger (backend) and browser (frontend)
- **Per wave merge:** `cd nemea-back && npx jest --config test/jest-e2e.json`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] E2E test file for products endpoints does not exist yet -- note: the project has NOT written e2e tests for previous modules (only app.e2e-spec.ts exists). Following the established project pattern, e2e tests are NOT created per module. Validation relies on manual Swagger testing + verification phase.
- [ ] Frontend test coverage: only one test file exists in `__tests__/page.test.tsx`. Frontend testing follows the minimal pattern established in the project.

**Note:** The project's established validation pattern is manual testing via Swagger (backend) and browser (frontend), plus the `/gsd:verify-work` verification phase. No per-module e2e test files have been created in phases 1-4.

## Sources

### Primary (HIGH confidence)
- Existing codebase: `nemea-back/src/supplies/` -- full module pattern (entity, service, controller, DTO, module)
- Existing codebase: `nemea-front/src/components/supplies/` -- full frontend pattern (table, group, expanded row, dialogs)
- Existing codebase: `nemea-back/src/database/migrations/` -- migration patterns
- Existing codebase: `nemea-back/src/common/entities/base.entity.ts` -- BaseEntity pattern
- CONTEXT.md: User decisions on SKU format, BOM versioning, batch creation, group editing

### Secondary (MEDIUM confidence)
- `db-model-draft.md` -- Original DB model reference for table structure
- Existing seed migration pattern from Phase 3

### Tertiary (LOW confidence)
- Costos Nemea.xlsx exact data values for seed -- needs extraction during execution

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already in project, no new deps needed
- Architecture: HIGH - follows established Supplies module pattern exactly
- Pitfalls: HIGH - identified from existing codebase patterns and known TypeORM behaviors
- BOM versioning: HIGH - straightforward transactional pattern with existing EntityManager
- Batch creation: MEDIUM - new pattern but well-defined by CONTEXT.md decisions
- BOM group editing: MEDIUM - majority detection algorithm is novel but straightforward

**Research date:** 2026-03-05
**Valid until:** 2026-04-05 (stable -- internal project, no external dependency changes)
