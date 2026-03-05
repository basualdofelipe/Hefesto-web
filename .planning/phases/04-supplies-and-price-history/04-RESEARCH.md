# Phase 4: Supplies and Price History - Research

**Researched:** 2026-03-05
**Domain:** NestJS CRUD with relational entities + append-only price history + grouped table UI
**Confidence:** HIGH

## Summary

Phase 4 builds two closely related entities (Supply and SupplyPriceHistory) on top of existing Phase 3 infrastructure (SupplyType, Supplier). The backend follows established NestJS module patterns (controller + service + entity + DTO + module) identical to the Suppliers module. The key architectural difference is the append-only price history pattern where "current price" is always the most recent record, requiring a composite index on `(supply_id, created_at DESC)`.

The frontend introduces a new UI pattern not seen in previous phases: a grouped expandable table with inline expand (not page navigation) plus modal dialogs for create/edit and price history. This builds on existing Shadcn/ui components (Table, Collapsible, Dialog) already installed in the project. The sidebar needs restructuring to add a "Datos base" group header using the existing `SidebarGroupLabel` component.

**Primary recommendation:** Follow the Suppliers module pattern exactly for the backend (entity, service, controller, DTOs, module, migration). Use TypeORM `@ManyToOne` relations for supply_type and supplier FKs, with eager loading on findAll. Price history is a separate entity with its own endpoints nested under `/supplies/:id/prices`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Grouped table by supply type with collapsible sections
- Click row expands inline (no page navigation)
- Expand shows: notas, proveedor link, precio actual with unit, ultima actualizacion, action buttons
- Supply data model: notes (text, nullable), unit_type (enum: m2, unidad, metro, kg), is_active with partial unique index on (name, supplier_id)
- Search box + supplier dropdown filter + "Mostrar inactivos" toggle
- Create/edit via modal dialog (not separate page)
- Proveedor NOT editable after creation
- Unique index on (name, supplier_id) WHERE is_active = true
- "+ Nuevo precio" opens inline input with confirm/cancel
- Price is optional at supply creation
- Price history is append-only (no edit/delete)
- Price history in modal: date + amount, newest first, no visual indicators
- Toggle switch for soft delete (same pattern as suppliers)
- Deactivating supplier cascades to deactivate its active supplies
- Reactivating supplier does NOT reactivate its supplies
- "Datos base" sidebar group header containing Catalogos, Proveedores, Insumos

### Claude's Discretion
- Exact styling of grouped table headers and collapsible animations
- Modal size and layout for create/edit and price history
- Loading and error states
- Toast messages and duration
- Search debounce timing
- Collapsible section default state (all open vs all closed)
- Empty state design for supplies list

### Deferred Ideas (OUT OF SCOPE)
- Structured supply specs by type (terminacion, espesor, colores for leather; dimensions for packaging)
- Visual price trend indicators (arrows, percentages, sparklines)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SPPL-01 | Admin can CRUD supplies with name, type, and supplier | Supply entity with FK relations to SupplyType and Supplier, SuppliesModule with CRUD service/controller, create/edit modal on frontend |
| SPPL-02 | Admin can add a new price to a supply (creates historical record, previous prices preserved) | SupplyPriceHistory entity, append-only POST endpoint, immutable records (no update/delete endpoints) |
| SPPL-03 | User can view price history for any supply | GET /supplies/:id/prices endpoint with ORDER BY created_at DESC, price history modal on frontend |
| SPPL-04 | Current price of a supply is always the most recent record in history | Composite index on (supply_id, created_at DESC), service method to get latest price, findAll returns supplies with current price via subquery or relation |
| SPPL-05 | Admin can deactivate/reactivate supplies (soft delete) | is_active column with partial unique index, PATCH toggle-status endpoint, supplier deactivation cascade |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/common | ^11.0.1 | NestJS framework | Project standard |
| @nestjs/typeorm | ^11.0.0 | TypeORM integration | Project standard |
| typeorm | ^0.3.28 | ORM with migrations | Project standard |
| class-validator | ^0.14.4 | DTO validation | Project standard |
| class-transformer | ^0.5.1 | DTO transformation | Project standard |
| @nestjs/swagger | ^11.2.6 | API documentation | Project standard |
| react-hook-form | ^7.71.2 | Form state management | Project standard |
| zod | ^4.3.6 | Schema validation | Project standard |
| @hookform/resolvers | ^5.2.2 | Zod + react-hook-form bridge | Project standard |
| sonner | ^2.0.7 | Toast notifications | Project standard |
| lucide-react | ^0.575.0 | Icons | Project standard |
| radix-ui | ^1.4.3 | UI primitives (via Shadcn) | Project standard |

### Shadcn/ui Components (already installed)
| Component | File | Purpose in This Phase |
|-----------|------|----------------------|
| Table | table.tsx | Supply list rows |
| Collapsible | collapsible.tsx | Expandable row sections by type |
| Dialog | dialog.tsx | Create/edit modal, price history modal |
| Input | input.tsx | Form fields, inline price input |
| Button | button.tsx | Actions |
| Select | select.tsx | Type, supplier, unit dropdowns |
| Switch | switch.tsx | Active/inactive toggle |
| Badge | badge.tsx | Status indicators |
| Label | label.tsx | Form labels |
| Skeleton | skeleton.tsx | Loading states |

**No new npm installs required.** All dependencies are already in both package.json files.

## Architecture Patterns

### Backend: Supply Entity with Relations

```
nemea-back/src/supplies/
  entities/
    supply.entity.ts           # Supply with ManyToOne to SupplyType + Supplier
    supply-price-history.entity.ts  # Append-only price records
  dto/
    create-supply.dto.ts       # name, typeId, supplierId, unitType, notes, initialPrice?
    update-supply.dto.ts       # name, typeId, unitType, notes (NOT supplierId)
    create-supply-price.dto.ts # price (decimal)
  supplies.controller.ts       # /supplies CRUD + /supplies/:id/prices
  supplies.service.ts          # Business logic + cascade deactivation
  supplies.module.ts           # Imports TypeOrmModule, SuppliersModule
```

### Pattern 1: TypeORM Entity with ManyToOne Relations
**What:** Supply entity references SupplyType and Supplier via `@ManyToOne` decorators
**When to use:** Every entity that has FK relationships
**Example:**
```typescript
// Follow same pattern as Supplier entity (base.entity.ts extension)
@Entity('supplies')
@Index('idx_supplies_name_supplier_active', ['name', 'supplier'], {
  where: '"is_active" = true',
  unique: true,
})
export class Supply extends BaseEntity {
  @Column({ type: 'varchar', length: 255, nullable: false })
  name!: string;

  @ManyToOne(() => SupplyType, { eager: true, nullable: false })
  @JoinColumn({ name: 'type_id' })
  type!: SupplyType;

  @ManyToOne(() => Supplier, { eager: true, nullable: false })
  @JoinColumn({ name: 'supplier_id' })
  supplier!: Supplier;

  @Column({ type: 'enum', enum: UnitType, default: UnitType.UNIDAD })
  unitType!: UnitType;

  @Column({ type: 'text', nullable: true })
  notes!: string | null;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}
```

### Pattern 2: Append-Only Price History
**What:** Immutable price records linked to a supply, ordered by creation date
**When to use:** Any entity where historical values must be preserved
**Example:**
```typescript
@Entity('supply_price_history')
@Index('idx_supply_price_history_supply_date', ['supply', 'createdAt'])
export class SupplyPriceHistory extends BaseEntity {
  @ManyToOne(() => Supply, { nullable: false, onDelete: 'CASCADE' })
  @JoinColumn({ name: 'supply_id' })
  supply!: Supply;

  @Column({ type: 'decimal', precision: 12, scale: 2, nullable: false })
  price!: number;
}
// Note: createdAt from BaseEntity serves as the timestamp
// Composite index on (supply_id, created_at) enables efficient latest-price queries
```

### Pattern 3: Partial Unique Index with Composite Key
**What:** Unique constraint on (name, supplier_id) only for active records
**Why:** Allows "Cuero Vacheta" from Supplier A and "Cuero Vacheta" from Supplier B simultaneously, while preventing duplicates for the same supplier
**Migration SQL:**
```sql
CREATE UNIQUE INDEX "idx_supplies_name_supplier_active"
  ON "supplies" ("name", "supplier_id")
  WHERE "is_active" = true;
```

### Pattern 4: Supplier Deactivation Cascade
**What:** When a supplier is deactivated, all its active supplies are also deactivated
**Implementation:** Modify `SuppliersService.toggleStatus` to cascade, or create a cross-module service
**Key decision:** SuppliersModule already exports SuppliersService. The SuppliesModule should import SuppliersModule and listen for/handle the cascade. However, the cleaner approach is to add cascade logic in SuppliesService and have SuppliersService call it (requires SuppliersModule to import SuppliesModule, creating a circular dep).

**Recommended approach:** Use `forwardRef` to break circular dependency, OR put cascade logic in a SupplierDeactivationSubscriber, OR more simply: have the frontend call both endpoints (toggle supplier, then bulk-deactivate supplies). **Best approach for this project:** Inject SuppliesService into SuppliersModule via forwardRef, since NestJS handles this cleanly.

**Alternative (simpler, recommended):** Have SuppliesModule export SuppliesService and SuppliersModule import SuppliesModule. Then SuppliersService.toggleStatus calls suppliesService.deactivateBySupplier() when deactivating. This avoids circular deps since Suppliers depends on Supplies (one direction only).

### Pattern 5: Current Price via Subquery
**What:** When listing supplies, include the current (latest) price without N+1 queries
**Options:**
1. **Subquery in findAll:** Use QueryBuilder to LEFT JOIN with a subquery that gets the latest price per supply
2. **Eager relation with limit:** Not supported by TypeORM for @OneToMany
3. **Separate query:** Fetch supplies, then batch-fetch latest prices

**Recommended:** Option 1 (subquery join) for the list endpoint, simple `find` with `ORDER BY created_at DESC LIMIT 1` for single supply detail.

```typescript
// For list: use QueryBuilder
const supplies = await this.supplyRepo
  .createQueryBuilder('supply')
  .leftJoinAndSelect('supply.type', 'type')
  .leftJoinAndSelect('supply.supplier', 'supplier')
  .leftJoin(
    (qb) => qb
      .select('DISTINCT ON (sph.supply_id) sph.supply_id', 'supply_id')
      .addSelect('sph.price', 'price')
      .addSelect('sph.created_at', 'created_at')
      .from(SupplyPriceHistory, 'sph')
      .orderBy('sph.supply_id')
      .addOrderBy('sph.created_at', 'DESC'),
    'latest_price',
    'latest_price.supply_id = supply.id',
  )
  .addSelect('latest_price.price', 'currentPrice')
  .addSelect('latest_price.created_at', 'lastPriceUpdate')
  .getRawAndEntities();
```

**Simpler alternative:** Fetch all supplies with their type/supplier relations using `find()`, then run a second query for latest prices grouped by supply_id. Map them in-service. This is cleaner code with 2 queries vs 1 complex query.

### Frontend: Grouped Expandable Table

```
nemea-front/src/
  app/(app)/insumos/
    page.tsx                    # Server component: fetch supplies + types + suppliers
  components/supplies/
    SupplyTable.tsx             # Client component: grouped table with expand
    SupplyTypeGroup.tsx         # Collapsible section per type
    SupplyExpandedRow.tsx       # Inline expanded content
    SupplyFormDialog.tsx        # Create/edit modal
    PriceHistoryDialog.tsx      # Price history modal
    AddPriceInline.tsx          # Inline price input with confirm/cancel
```

### Anti-Patterns to Avoid
- **N+1 queries for current price:** Never fetch each supply's latest price individually in a loop
- **Mutable price records:** Never add update/delete endpoints for price history
- **Eager loading @OneToMany:** TypeORM's eager on OneToMany loads ALL history records -- never do this on the supply entity
- **Editing supplier on existing supply:** Violates the immutable supply-supplier relationship decision

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Form validation | Custom validation logic | react-hook-form + zod + zodResolver | Already established pattern in SupplierForm |
| UUID validation | Manual regex | ParseUUIDPipe (NestJS built-in) | Already used in SuppliersController |
| Unique constraint errors | Try/catch with raw SQL error codes | Catch QueryFailedError code 23505 | Already established pattern in SuppliersService |
| FK violation detection | Manual pre-check queries | Catch QueryFailedError code 23503 | Already established pattern in CatalogsService |
| Toast notifications | Custom notification system | sonner toast() | Already wired in the project |
| Collapsible sections | Custom expand/collapse state | Shadcn Collapsible component | Already installed |
| Modal dialogs | Custom overlay implementation | Shadcn Dialog component | Already installed |
| Decimal handling | JavaScript floating point | PostgreSQL DECIMAL(12,2) + string transport | Avoids precision loss |

## Common Pitfalls

### Pitfall 1: TypeORM Partial Index in Entity Decorator
**What goes wrong:** TypeORM's `@Index` decorator with composite columns and a WHERE clause can be tricky with relation columns
**Why it happens:** When using `@ManyToOne`, the actual DB column name (supplier_id) differs from the entity property name (supplier)
**How to avoid:** Use the raw column name in the `@Index` where clause: `where: '"is_active" = true'`. For the composite unique index, define it in the migration SQL directly rather than relying on the entity decorator, since the decorator may not correctly resolve the FK column name for composite indexes.
**Warning signs:** Migration generates unexpected index SQL

### Pitfall 2: Decimal Precision Loss
**What goes wrong:** JavaScript treats DECIMAL as number, losing precision for large values
**Why it happens:** TypeORM returns DECIMAL columns as strings by default in some drivers, but may coerce to number
**How to avoid:** Use `DECIMAL(12,2)` in PostgreSQL (sufficient for ARS prices up to 9,999,999,999.99). When returning to the frontend, the response envelope will contain the price as a number. This is acceptable for ARS amounts in this project scope.
**Warning signs:** Prices display as 15200.000000001 in the UI

### Pitfall 3: Cascade Deactivation Ordering
**What goes wrong:** Deactivating a supplier should cascade to supplies, but reactivation should NOT cascade
**Why it happens:** Symmetric cascade logic applied in both directions
**How to avoid:** In the toggle logic: `if (!newIsActive)` -> deactivate all active supplies for this supplier. If reactivating, do nothing to supplies.
**Warning signs:** Reactivating a supplier magically reactivates supplies the admin intentionally left inactive

### Pitfall 4: Missing Relations in API Response
**What goes wrong:** Supply list returns type_id and supplier_id UUIDs instead of the actual type/supplier objects
**Why it happens:** TypeORM doesn't join relations by default unless `eager: true` or `relations: [...]` is specified
**How to avoid:** Use `eager: true` on `@ManyToOne` for type and supplier relations in the Supply entity, OR always specify `relations: ['type', 'supplier']` in find options
**Warning signs:** Frontend receives UUIDs instead of objects for type/supplier

### Pitfall 5: Collapsible State Management
**What goes wrong:** Expanding one supply type group collapses others, or all groups re-render on any change
**Why it happens:** Shared state between collapsible components, or unnecessary re-renders from parent state changes
**How to avoid:** Each `SupplyTypeGroup` manages its own open/closed state independently. Use `useState` per group, not a single state object in the parent.
**Warning signs:** UI feels janky when expanding/collapsing groups

### Pitfall 6: Price Optional at Creation but Required in History
**What goes wrong:** Creating a supply with an initial price requires both the supply and the price record to be created, but they're in different tables
**Why it happens:** Two-step operation (create supply, then create price) needs to be atomic
**How to avoid:** In `SuppliesService.create()`, if `initialPrice` is provided in the DTO, create both the supply and the price record in a single transaction using `queryRunner` or `entityManager.transaction()`
**Warning signs:** Supply created but price record missing on error

### Pitfall 7: SuppliersModule Needs to Export Service for Cascade
**What goes wrong:** SuppliesService needs to be injectable in SuppliersModule for cascade deactivation
**Why it happens:** Circular dependency: Suppliers needs Supplies (for cascade), Supplies needs Suppliers (for FK)
**How to avoid:** Direction of dependency should be: SuppliesModule imports SuppliersModule (for FK validation). For cascade: override SuppliersService.toggleStatus in the SuppliesModule by having SuppliesModule provide a listener or by using an event-based approach. **Simplest approach:** Just do the cascade from the frontend (two API calls) or from SuppliersService by directly querying the supplies table via TypeORM repository (inject Supply repository in SuppliersModule via TypeOrmModule.forFeature).
**Warning signs:** NestJS circular dependency error on startup

## Code Examples

### Backend: Supply Entity
```typescript
// Source: Project patterns from supplier.entity.ts + db-model-draft.md
import { Column, Entity, Index, JoinColumn, ManyToOne } from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';
import { SupplyType } from '../../catalogs/entities/supply-type.entity';
import { Supplier } from '../../suppliers/entities/supplier.entity';

export enum UnitType {
  M2 = 'm2',
  UNIDAD = 'unidad',
  METRO = 'metro',
  KG = 'kg',
}

@Entity('supplies')
export class Supply extends BaseEntity {
  @Column({ type: 'varchar', length: 255, nullable: false })
  name!: string;

  @ManyToOne(() => SupplyType, { eager: true, nullable: false })
  @JoinColumn({ name: 'type_id' })
  type!: SupplyType;

  @ManyToOne(() => Supplier, { eager: true, nullable: false })
  @JoinColumn({ name: 'supplier_id' })
  supplier!: Supplier;

  @Column({ name: 'unit_type', type: 'enum', enum: UnitType, default: UnitType.UNIDAD })
  unitType!: UnitType;

  @Column({ type: 'text', nullable: true })
  notes!: string | null;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}
```

### Backend: SupplyPriceHistory Entity
```typescript
// Source: Project patterns from base.entity.ts + db-model-draft.md
import { Column, Entity, JoinColumn, ManyToOne } from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';
import { Supply } from './supply.entity';

@Entity('supply_price_history')
export class SupplyPriceHistory extends BaseEntity {
  @ManyToOne(() => Supply, { nullable: false, onDelete: 'CASCADE' })
  @JoinColumn({ name: 'supply_id' })
  supply!: Supply;

  @Column({ type: 'decimal', precision: 12, scale: 2, nullable: false })
  price!: string; // TypeORM returns decimal as string
}
// Index created in migration: (supply_id, created_at DESC)
```

### Backend: Create Supply DTO
```typescript
// Source: Project patterns from create-supplier.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEnum, IsNotEmpty, IsNumber, IsOptional, IsString, IsUUID, MaxLength, Min } from 'class-validator';
import { UnitType } from '../entities/supply.entity';

export class CreateSupplyDto {
  @ApiProperty({ description: 'Nombre del insumo', example: 'Cuero Vacheta' })
  @IsString()
  @IsNotEmpty({ message: 'El nombre es obligatorio' })
  @MaxLength(255)
  name!: string;

  @ApiProperty({ description: 'ID del tipo de insumo' })
  @IsUUID()
  @IsNotEmpty({ message: 'El tipo es obligatorio' })
  typeId!: string;

  @ApiProperty({ description: 'ID del proveedor' })
  @IsUUID()
  @IsNotEmpty({ message: 'El proveedor es obligatorio' })
  supplierId!: string;

  @ApiProperty({ description: 'Unidad de medida', enum: UnitType })
  @IsEnum(UnitType, { message: 'Unidad de medida invalida' })
  unitType!: UnitType;

  @ApiProperty({ required: false, description: 'Notas del insumo' })
  @IsString()
  @IsOptional()
  notes?: string;

  @ApiProperty({ required: false, description: 'Precio inicial (opcional)' })
  @IsNumber({}, { message: 'El precio debe ser un numero' })
  @Min(0, { message: 'El precio no puede ser negativo' })
  @IsOptional()
  initialPrice?: number;
}
```

### Backend: Migration SQL Pattern
```sql
-- supplies table
CREATE TABLE "supplies" (
  "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  "name" character varying(255) NOT NULL,
  "type_id" uuid NOT NULL,
  "supplier_id" uuid NOT NULL,
  "unit_type" "public"."supplies_unit_type_enum" NOT NULL DEFAULT 'unidad',
  "notes" text,
  "is_active" boolean NOT NULL DEFAULT true,
  CONSTRAINT "PK_supplies_id" PRIMARY KEY ("id"),
  CONSTRAINT "FK_supplies_type" FOREIGN KEY ("type_id") REFERENCES "supply_types"("id"),
  CONSTRAINT "FK_supplies_supplier" FOREIGN KEY ("supplier_id") REFERENCES "suppliers"("id")
);

CREATE UNIQUE INDEX "idx_supplies_name_supplier_active"
  ON "supplies" ("name", "supplier_id") WHERE "is_active" = true;

-- supply_price_history table
CREATE TABLE "supply_price_history" (
  "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  "supply_id" uuid NOT NULL,
  "price" decimal(12,2) NOT NULL,
  CONSTRAINT "PK_supply_price_history_id" PRIMARY KEY ("id"),
  CONSTRAINT "FK_supply_price_history_supply" FOREIGN KEY ("supply_id") REFERENCES "supplies"("id") ON DELETE CASCADE
);

CREATE INDEX "idx_supply_price_history_supply_date"
  ON "supply_price_history" ("supply_id", "created_at" DESC);
```

### Frontend: Zod Schema for Supply Form
```typescript
// Source: Project patterns from SupplierForm.tsx
import { z } from 'zod';

const supplySchema = z.object({
  name: z.string().min(1, 'El nombre es obligatorio').max(255),
  typeId: z.string().uuid('Selecciona un tipo'),
  supplierId: z.string().uuid('Selecciona un proveedor'),
  unitType: z.enum(['m2', 'unidad', 'metro', 'kg']),
  notes: z.string().optional().or(z.literal('')),
  initialPrice: z.number().min(0).optional(),
});
```

### API Endpoints
```
GET    /api/supplies                    # List all (with current price, grouped by type)
GET    /api/supplies/:id                # Single supply detail
POST   /api/supplies                    # Create supply (ADMIN)
PUT    /api/supplies/:id                # Update supply (ADMIN)
PATCH  /api/supplies/:id/toggle-status  # Toggle active/inactive (ADMIN)
GET    /api/supplies/:id/prices         # Price history (newest first)
POST   /api/supplies/:id/prices         # Add new price (ADMIN)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate detail pages (proveedores pattern) | Inline expand in table + modals | Phase 4 (new) | Keeps user in list context |
| Flat table list | Grouped table by type with collapsible sections | Phase 4 (new) | Better information architecture |
| Simple CRUD | CRUD + append-only history pattern | Phase 4 (new) | Foundation for Phase 6 cost calculation |

## Open Questions

1. **Enum registration in PostgreSQL**
   - What we know: TypeORM handles `type: 'enum'` automatically, creating a PostgreSQL enum type
   - What's unclear: The auto-generated enum name (e.g., `supplies_unit_type_enum`) may vary. Migration should create the enum explicitly for reproducibility.
   - Recommendation: Let TypeORM generate the migration, then verify the enum name is stable. If using raw SQL migration, create the enum type explicitly before the table.

2. **Circular dependency for cascade deactivation**
   - What we know: Supplier toggle needs to cascade to supplies, but Supplies already depends on Suppliers
   - What's unclear: Best pattern for this specific project
   - Recommendation: Add Supply repository directly to SuppliersModule via `TypeOrmModule.forFeature([Supply])` and handle cascade in SuppliersService. This avoids circular module deps while keeping the logic server-side. SuppliersModule would import `TypeOrmModule.forFeature([Supply])` directly (not SuppliesModule).

3. **Price decimal handling in frontend**
   - What we know: TypeORM returns DECIMAL as string, NestJS response envelope may keep or convert it
   - What's unclear: Whether class-transformer will convert price string to number automatically
   - Recommendation: Accept string from backend, parse with `parseFloat()` on frontend for display. Use `toLocaleString('es-AR')` for currency formatting.

## Sources

### Primary (HIGH confidence)
- Project codebase: `nemea-back/src/suppliers/` -- complete module pattern reference
- Project codebase: `nemea-back/src/common/entities/base.entity.ts` -- BaseEntity pattern
- Project codebase: `nemea-back/src/database/migrations/` -- migration SQL pattern
- Project codebase: `nemea-front/src/components/suppliers/` -- UI component patterns
- Project codebase: `nemea-front/src/lib/api.ts` + `api-client.ts` -- API fetch patterns
- 04-CONTEXT.md -- All user decisions and constraints

### Secondary (MEDIUM confidence)
- db-model-draft.md -- Original DB model for supplies and supply_price_history tables
- TypeORM documentation for @ManyToOne, @Index with where clause, enum columns

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already installed and used in previous phases
- Architecture: HIGH - follows established patterns from Phase 3 with clear extensions
- Pitfalls: HIGH - identified from direct codebase analysis and TypeORM known behaviors
- Frontend UI pattern: MEDIUM - new grouped/expandable pattern not yet implemented, but uses installed components

**Research date:** 2026-03-05
**Valid until:** 2026-04-05 (stable -- no library changes expected)
