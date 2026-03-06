---
phase: 05-products-and-bom
verified: 2026-03-06T02:10:00Z
status: passed
score: 5/5 must-haves verified
---

# Phase 5: Products and BOM Verification Report

**Phase Goal:** Products exist with auto-generated SKUs, their material composition is defined and versioned, and their selling price is tracked
**Verified:** 2026-03-06T02:10:00Z
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths (from ROADMAP Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Admin can create a product by selecting type, name, finish, color, and size -- a unique SKU is generated automatically | VERIFIED | `products.service.ts` has `create()` with `loadDimensions()` + `generateSkuCode()` producing dot-separated format. Frontend `ProductBatchCreateDialog.tsx` has 4-step wizard (340+ lines). Backend `CreateProductDto` requires 5 UUID dimension IDs. |
| 2 | Admin can define the BOM (list of supplies with quantity and unit_type) for a product | VERIFIED | `products.service.ts` has `updateBom()` with transactional version swap. `BomEditorDialog.tsx` (260+ lines) has supply Combobox via Command component, quantity input, unit auto-fill. API: `PUT /products/:id/bom`. |
| 3 | When Admin changes the BOM, the old composition is preserved with is_active=false and the new one becomes active | VERIFIED | `products.service.ts` lines 296-301: `manager.update(SuppliesPerProductHistory, { product: { id: productId }, isActive: true }, { isActive: false })` followed by INSERT of new entries, all within `entityManager.transaction()`. |
| 4 | Admin can add a selling price to a product; previous selling prices are preserved as history | VERIFIED | `products.service.ts` has `addPrice()` (append-only to `product_price_history`), `getPriceHistory()` ordered by createdAt DESC. Frontend: `AddSellingPriceInline.tsx` POSTs to `/api/products/:id/prices`, `PriceHistoryDialog.tsx` fetches and displays history. |
| 5 | Admin can deactivate a product; it is excluded from active product views but the record is not deleted | VERIFIED | `products.service.ts` has `toggleStatus()` flipping `isActive`. `findAll()` filters `where: { isActive: true }` unless `includeInactive`. Frontend: toggle button in `ProductExpandedRow.tsx` calls `PATCH /products/:id/toggle-status`. |

**Score:** 5/5 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `nemea-back/src/products/entities/product.entity.ts` | Product entity with 5 FK dimensions + skuCode + isActive | VERIFIED | 36 lines, 5 ManyToOne relations, skuCode varchar, isActive boolean |
| `nemea-back/src/products/products.service.ts` | Product CRUD with SKU generation and batch creation + BOM + prices | VERIFIED | 485 lines, generateSkuCode, create, createBatch, update, toggleStatus, getBom, updateBom, batchUpdateBom, getPriceHistory, addPrice, batchAddPrice |
| `nemea-back/src/products/products.controller.ts` | REST endpoints for product CRUD + BOM + prices | VERIFIED | 264 lines, 12 endpoints with Swagger decorators, Roles guards, batch routes before :id routes |
| `nemea-back/src/products/products.module.ts` | Module with TypeORM repos, exports ProductsService | VERIFIED | 33 lines, 9 entities in forFeature, exports ProductsService |
| `nemea-back/src/products/entities/supplies-per-product-history.entity.ts` | BOM entity with product FK, supply FK, quantity, is_active | VERIFIED | 21 lines, ManyToOne Product/Supply, decimal quantity, isActive |
| `nemea-back/src/products/entities/product-price-history.entity.ts` | Product price history with decimal price, currency | VERIFIED | 21 lines, ManyToOne Product, decimal price, currency default ARS |
| `nemea-back/src/database/migrations/1772500000000-AddSkuCodeToCatalogs.ts` | sku_code column on 5 catalog dimension tables | VERIFIED | Contains sku_code column additions with ROW_NUMBER fallback |
| `nemea-back/src/database/migrations/1772500100000-CreateProductsTable.ts` | Products table with FKs and partial unique index | VERIFIED | CREATE TABLE products + UQ_products_sku_code_active partial index |
| `nemea-back/src/database/migrations/1772500200000-CreateBomAndPriceHistory.ts` | BOM and price history tables | VERIFIED | CREATE TABLE supplies_per_product_history + product_price_history with indexes |
| `nemea-back/src/database/migrations/1772500300000-SeedProductsAndBom.ts` | Seed products, BOM, prices | VERIFIED | 12499 bytes of seed data |
| `nemea-front/src/app/(app)/productos/page.tsx` | Server component fetching products + catalogs + supplies | VERIFIED | Promise.all with 7 parallel fetches, passes to ProductTable |
| `nemea-front/src/components/products/ProductTable.tsx` | Main product table with search, filter, grouped rendering | VERIFIED | Renders ProductTypeGroup components, search input, show-inactive toggle |
| `nemea-front/src/components/products/ProductBatchCreateDialog.tsx` | Multi-step batch creation modal | VERIFIED | 4-step wizard (step 0-3), react-hook-form + zod, checkbox grids, SKU preview |
| `nemea-front/src/components/products/ProductExpandedRow.tsx` | Expanded row with BOM, price, action buttons | VERIFIED | Fetches BOM on expand, renders all action buttons wired to dialogs |
| `nemea-front/src/components/products/BomEditorDialog.tsx` | BOM editor modal with supply Combobox | VERIFIED | CommandInput/CommandList/CommandGroup, quantity input, unit auto-fill |
| `nemea-front/src/components/products/BomGroupEditorDialog.tsx` | Group BOM editor with majority detection | VERIFIED | serializeBom + JSON.stringify comparison, divergent product warnings, batch-bom API call |
| `nemea-front/src/components/products/AddSellingPriceInline.tsx` | Inline price input with confirm/cancel | VERIFIED | POSTs to /api/products/:id/prices with toast feedback |
| `nemea-front/src/components/products/PriceHistoryDialog.tsx` | Price history modal | VERIFIED | Fetches /api/products/:id/prices, displays date + amount list |
| `nemea-front/src/components/products/BatchPriceDialog.tsx` | Batch price update dialog | VERIFIED | Product checkboxes, single price input, POSTs to /api/products/batch-prices |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| products.service.ts | product.entity.ts | TypeORM repository | WIRED | `@InjectRepository(Product)` + full CRUD operations |
| products.module.ts | app.module.ts | Module registration | WIRED | `ProductsModule` imported in app.module.ts line 33 |
| products.service.ts | supplies-per-product-history.entity.ts | EntityManager.transaction for BOM version swap | WIRED | `manager.update(...isActive: false)` + `manager.save(newEntries)` in transaction |
| supplies.service.ts | supplies-per-product-history.entity.ts | BOM usage check before supply deactivation | WIRED | `bomRepo.count({ where: { supply: { id }, isActive: true } })` blocks deactivation |
| BomEditorDialog.tsx | /api/products/:id/bom | apiClientFetch PUT | WIRED | Line 200: `apiClientFetch(/api/products/${productId}/bom, ...)` |
| BomGroupEditorDialog.tsx | /api/products/batch-bom | apiClientFetch PUT | WIRED | Line 283: `apiClientFetch('/api/products/batch-bom', ...)` |
| ProductExpandedRow.tsx | BomEditorDialog + PriceHistoryDialog + AddSellingPriceInline | Component imports + rendering | WIRED | All 3 imported and rendered with props |
| ProductTypeGroup.tsx | BomGroupEditorDialog + BatchPriceDialog | Component imports + rendering | WIRED | Both imported and rendered with props |
| productos/page.tsx | /api/products | apiFetch server-side | WIRED | `apiFetch<{ data: Product[] }>('/api/products?includeInactive=true')` |
| AppSidebar.tsx | /productos | Nav link | WIRED | `{ label: 'Productos', href: '/productos', icon: ShoppingBag }` |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| PROD-01 | 05-01, 05-03 | Admin can create a product by selecting type, name, finish, color, and size | SATISFIED | Backend create() + frontend ProductBatchCreateDialog 4-step wizard |
| PROD-02 | 05-01, 05-03 | Each product has an auto-generated unique SKU (sku_code) | SATISFIED | generateSkuCode() produces dot-separated format, partial unique index on active products |
| PROD-03 | 05-01, 05-03 | Admin can edit product attributes | SATISFIED | Backend update() + frontend ProductEditDialog with live SKU preview |
| PROD-04 | 05-01, 05-03 | Admin can deactivate/reactivate products (soft delete) | SATISFIED | Backend toggleStatus() + frontend toggle button + findAll filter |
| PROD-05 | 05-02, 05-04 | Admin can define material composition (BOM) with quantity and unit_type | SATISFIED | BOM entity + updateBom() transactional swap + BomEditorDialog with supply Combobox |
| PROD-06 | 05-02, 05-04 | When BOM changes, old composition is preserved as history | SATISFIED | Version swap: UPDATE is_active=false + INSERT new, all in transaction |
| PROD-07 | 05-02, 05-04 | Admin can add a selling price to a product (historical record) | SATISFIED | ProductPriceHistory append-only + AddSellingPriceInline + PriceHistoryDialog |

No orphaned requirements found. All 7 PROD requirements are covered by plans and verified in codebase.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none) | - | - | - | No anti-patterns detected |

No TODOs, FIXMEs, stubs, console.logs, placeholder returns, or empty handlers found in any phase 5 files.

### Human Verification Required

### 1. Full Product Lifecycle Flow

**Test:** Navigate to /productos, create products via batch wizard, expand a product, add BOM, add selling price, edit product dimensions, deactivate/reactivate
**Expected:** All operations complete successfully with toast feedback, data persists on page refresh
**Why human:** End-to-end flow across multiple dialogs and API calls requires visual confirmation

### 2. BOM Group Editor Majority Detection

**Test:** Set different BOMs on products of the same type, then open group BOM editor from type header
**Expected:** Products with non-majority BOM show warning indicator (yellow triangle), majority BOM pre-populates editor
**Why human:** Visual indicator placement and majority detection behavior requires interactive testing

### 3. Supply Deactivation Guard

**Test:** Add a supply to a product's BOM, then try to deactivate that supply from the /insumos page
**Expected:** Error toast appears with message about supply being in use by N product(s)
**Why human:** Cross-page interaction between supplies and products modules

### 4. Migration Robustness

**Test:** Start backend with fresh database (or with user-created catalog items not in seed data)
**Expected:** All 4 migrations run without errors, sku_code assigned to all catalog items including user-created ones
**Why human:** Database state-dependent behavior, ROW_NUMBER fallback for dynamic catalog items

### Gaps Summary

No gaps found. All 5 success criteria from ROADMAP are verified in the codebase. All 7 PROD requirements are satisfied. Backend has 12 REST endpoints (6 product CRUD + 2 BOM + 2 price + 2 batch) with full Swagger decorators. Frontend has 11 components providing complete product management UI with batch creation, BOM editing (single + group), selling price management (inline + history + batch), and toggle status. Supply deactivation guard prevents data integrity violations. All key links are wired -- no orphaned components.

---

_Verified: 2026-03-06T02:10:00Z_
_Verifier: Claude (gsd-verifier)_
