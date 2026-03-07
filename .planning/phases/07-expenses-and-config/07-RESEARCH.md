# Phase 7: Expenses - Research

**Researched:** 2026-03-06
**Domain:** Expense CRUD with categories, grouped list, filtering
**Confidence:** HIGH

## Summary

Phase 7 implements expense tracking -- a straightforward CRUD module with two entities (Expense, ExpenseCategory) following patterns already established across 6 prior phases. The backend needs a new ExpensesModule with standard NestJS controller/service/entity/DTO/module plus the ExpenseCategory entity integrated into the existing CatalogsModule. The frontend needs a new `/finanzas/gastos` route with a grouped-by-month expense table (adapting the SupplyTypeGroup collapsible pattern), a create/edit dialog (adapting the SupplyFormDialog pattern), and a new "Categorias de Gasto" tab on the existing `/catalogos` page (reusing CatalogTabContent verbatim via the dimension map).

The entire phase reuses existing patterns with zero new libraries needed. The only novel UI element is the month-grouped collapsible list with a dynamic summary bar and date range filtering -- all achievable with existing Shadcn/ui components (Collapsible, Badge, Table, Dialog, Select) and react-hook-form + zod.

**Primary recommendation:** Follow established module patterns exactly. ExpenseCategory as a new catalog dimension (added to CatalogsService dimension map). Expense as a standalone module with its own controller/service. Frontend expense list adapts SupplyTypeGroup for month grouping.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Expense list grouped by month with collapsible sections, each header showing month name + count + subtotal
- Table columns: Fecha, Concepto, Categoria (colored badge), Monto
- Summary bar at top with grand total + count + filtered period, updates dynamically
- Default view: current month only
- Category dropdown filter (one or "Todas") + date range picker (desde/hasta), both combine
- Each of 6 categories gets a distinct colored badge
- Create/edit via modal dialog (same pattern as supply create in Phase 4)
- Fields: monto (required), concepto (required, free text), fecha (required, defaults today), categoria (required, dropdown)
- Hard delete from DB with confirmation dialog
- Categories stored in DB as catalog entity, NOT hardcoded enum
- New tab "Categorias de Gasto" on existing /catalogos page with inline CRUD
- Seeded with 6 initial categories: materia prima, packaging, envio, herramientas, servicios, otros
- Block deletion if category is used by existing expenses
- New sidebar group "Finanzas" with "Gastos" item
- Sidebar order: Inicio > Finanzas (Gastos) > Productos > Datos base (Catalogos, Proveedores, Insumos)
- Route: /finanzas/gastos
- ADMIN: full CRUD; USER: read-only
- Tiendanube config REMOVED from v1

### Claude's Discretion
- Exact styling of month group headers and collapsible animations
- Modal size and layout for expense create/edit
- Category badge color assignments
- Loading and error states
- Toast messages and duration
- Date picker component choice and behavior
- Empty state design for expenses list
- Summary bar styling and layout

### Deferred Ideas (OUT OF SCOPE)
- Tiendanube config (CONF-01, CONF-02) -- v2, alongside calculadora
- Expense reports/analytics (totals by category over time, charts) -- v2 dashboard
- Expense export to Excel/PDF -- v2
- Recurring expenses (auto-create monthly) -- v2+
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| EXPN-01 | Admin can register an expense with amount, concept, date, and category | Expense entity with decimal amount, varchar concept, timestamptz date, FK to expense_categories. ExpenseFormDialog with react-hook-form + zod. ADMIN-only write endpoints. |
| EXPN-02 | Admin can view list of expenses filtered by category or date | GET /api/expenses with optional query params (categoryId, dateFrom, dateTo). Frontend month-grouped collapsible list with filter controls. Summary bar with dynamic totals. |
| EXPN-03 | Categories include: materia prima, packaging, envio, herramientas, servicios, otros | ExpenseCategory entity added to CatalogsModule dimension map. Seed migration with 6 initial categories. Inline CRUD on /catalogos page as new tab. FK constraint blocks delete when referenced. |
</phase_requirements>

## Standard Stack

### Core (already installed -- no new dependencies)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Backend framework | Already in use |
| TypeORM | 0.3.x | ORM + migrations | Already in use |
| Next.js | 16.x | Frontend framework | Already in use |
| react-hook-form | 7.x | Form state management | Already in use for supplies |
| zod | 3.x | Schema validation | Already in use |
| @hookform/resolvers | 3.x | Zod resolver for react-hook-form | Already in use |
| sonner | latest | Toast notifications | Already wired |
| Shadcn/ui | latest | UI components (Dialog, Table, Badge, Collapsible, Select, Input, Button) | Already in use |
| lucide-react | latest | Icons | Already in use |

### No New Libraries Needed
This phase adds zero new dependencies. All functionality is covered by existing stack.

### Date Picker
Shadcn/ui does not include a built-in date range picker as a single component. Two approaches:

| Approach | Complexity | UX |
|----------|-----------|-----|
| Two `<Input type="date">` fields | Minimal -- native HTML | Functional, consistent cross-browser on desktop |
| Shadcn/ui `Calendar` + `Popover` | Medium -- needs manual wiring | Prettier, matches app aesthetic |

**Recommendation:** Use native `<Input type="date">` for the filter date range (desde/hasta) and for the expense form date field. It is the simplest approach, avoids new dependencies (date-fns, react-day-picker), and the app is admin-only desktop usage. This aligns with "funcionalidad primero, iterar diseno despues."

## Architecture Patterns

### Backend: New Module Structure
```
nemea-back/src/
├── expenses/
│   ├── expenses.module.ts
│   ├── expenses.controller.ts
│   ├── expenses.service.ts
│   ├── entities/
│   │   └── expense.entity.ts
│   └── dto/
│       ├── create-expense.dto.ts
│       ├── update-expense.dto.ts
│       └── query-expenses.dto.ts
├── catalogs/
│   ├── entities/
│   │   └── expense-category.entity.ts  ← NEW entity
│   ├── catalogs.service.ts             ← add 'expense-categories' dimension
│   ├── catalogs.controller.ts          ← add to DIMENSION_ENUM
│   └── catalogs.module.ts              ← register ExpenseCategory repo
```

### Backend: Expense Entity Design
```typescript
@Entity('expenses')
export class Expense extends BaseEntity {
  @Column({ type: 'decimal', precision: 12, scale: 2, nullable: false })
  amount!: string; // TypeORM decimal → string (same as SupplyPriceHistory.price)

  @Column({ type: 'varchar', length: 500, nullable: false })
  concept!: string;

  @Column({ type: 'date', nullable: false })
  date!: string; // DATE type (not timestamptz) -- expenses are date-level, no time

  @ManyToOne(() => ExpenseCategory, { eager: true, nullable: false })
  @JoinColumn({ name: 'category_id' })
  category!: ExpenseCategory;
}
```

**Key design decisions:**
- `amount` as `decimal(12,2)` -- same pattern as SupplyPriceHistory.price (TypeORM returns string, frontend parses with parseFloat)
- `date` as PostgreSQL `DATE` type (not timestamptz) -- expenses are tracked by calendar date, time is irrelevant
- `category` as eager ManyToOne -- category name needed in every list row, small cardinality
- NO `is_active` -- hard delete per user decision

### Backend: ExpenseCategory Entity
```typescript
@Entity('expense_categories')
export class ExpenseCategory extends BaseEntity {
  @Column({ type: 'varchar', length: 100, unique: true })
  name!: string;
}
```
Identical pattern to SupplyType. Registered in CatalogsModule + CatalogsService dimension map as `'expense-categories'`.

### Backend: API Endpoints
```
GET    /api/expenses?categoryId=uuid&dateFrom=2026-01-01&dateTo=2026-03-31
POST   /api/expenses          (ADMIN only)
PUT    /api/expenses/:id      (ADMIN only)
DELETE /api/expenses/:id      (ADMIN only)

GET    /api/catalogs/expense-categories
POST   /api/catalogs/expense-categories          (ADMIN only)
PUT    /api/catalogs/expense-categories/:id      (ADMIN only)
DELETE /api/catalogs/expense-categories/:id      (ADMIN only)
```

The GET /api/expenses endpoint should return expenses ordered by `date DESC, createdAt DESC`. The frontend handles month grouping client-side (grouping a flat list by `YYYY-MM` from the date field).

### Backend: Query Filtering Pattern
```typescript
async findAll(query: QueryExpensesDto): Promise<Expense[]> {
  const qb = this.expenseRepo.createQueryBuilder('expense')
    .leftJoinAndSelect('expense.category', 'category')
    .orderBy('expense.date', 'DESC')
    .addOrderBy('expense.createdAt', 'DESC');

  if (query.categoryId) {
    qb.andWhere('expense.category_id = :categoryId', { categoryId: query.categoryId });
  }
  if (query.dateFrom) {
    qb.andWhere('expense.date >= :dateFrom', { dateFrom: query.dateFrom });
  }
  if (query.dateTo) {
    qb.andWhere('expense.date <= :dateTo', { dateTo: query.dateTo });
  }

  return qb.getMany();
}
```

### Frontend: Route Structure
```
nemea-front/src/app/(app)/
├── finanzas/
│   └── gastos/
│       └── page.tsx          ← server component, fetches expenses + categories
├── catalogos/
│   └── page.tsx              ← add 'expense-categories' to DIMENSIONS array
```

### Frontend: Component Structure
```
nemea-front/src/components/
├── expenses/
│   ├── types.ts              ← Expense, ExpenseCategory interfaces, formatAmount, CATEGORY_COLORS
│   ├── ExpenseTable.tsx      ← client component: filters + month groups + summary bar
│   ├── ExpenseMonthGroup.tsx ← collapsible month group (adapts SupplyTypeGroup)
│   ├── ExpenseFormDialog.tsx ← create/edit modal (adapts SupplyFormDialog)
│   └── DeleteExpenseDialog.tsx ← confirmation dialog for hard delete
```

### Frontend: Month Grouping (Client-side)
```typescript
// Group expenses by YYYY-MM from the date field
function groupByMonth(expenses: Expense[]): Map<string, Expense[]> {
  const groups = new Map<string, Expense[]>();
  for (const expense of expenses) {
    const key = expense.date.substring(0, 7); // "2026-03"
    const existing = groups.get(key) ?? [];
    existing.push(expense);
    groups.set(key, existing);
  }
  return groups;
}

// Format month key to display: "Marzo 2026"
function formatMonthLabel(key: string): string {
  const [year, month] = key.split('-');
  const date = new Date(parseInt(year), parseInt(month) - 1);
  return date.toLocaleDateString('es-AR', { month: 'long', year: 'numeric' });
}
```

### Frontend: Category Badge Colors
```typescript
const CATEGORY_COLORS: Record<string, string> = {
  'materia prima': 'bg-amber-100 text-amber-800',
  'packaging': 'bg-blue-100 text-blue-800',
  'envio': 'bg-green-100 text-green-800',
  'herramientas': 'bg-purple-100 text-purple-800',
  'servicios': 'bg-cyan-100 text-cyan-800',
  'otros': 'bg-gray-100 text-gray-800',
};

// Fallback for user-created categories
function getCategoryColor(name: string): string {
  return CATEGORY_COLORS[name.toLowerCase()] ?? 'bg-gray-100 text-gray-800';
}
```

### Frontend: Sidebar Restructure
Current sidebar groups:
1. (no label): Inicio
2. "Datos base": Catalogos, Proveedores, Insumos, Productos

New sidebar groups:
1. (no label): Inicio
2. "Finanzas": Gastos (icon: DollarSign or Receipt from lucide-react)
3. (no label): Productos (moved out of Datos base to match the order)
4. "Datos base": Catalogos, Proveedores, Insumos

Wait -- per CONTEXT.md the order is: Inicio > Finanzas (Gastos) > Productos > Datos base (Catalogos, Proveedores, Insumos). This means "Productos" is no longer inside "Datos base" -- it becomes its own top-level item between Finanzas and Datos base.

### Anti-Patterns to Avoid
- **Don't aggregate in the backend for month grouping:** The frontend groups by month client-side. The backend returns a flat list. This keeps the API simple and reusable.
- **Don't use timestamptz for expense date:** Expenses are calendar dates. Using timestamptz would introduce timezone display issues. Use PostgreSQL `DATE` type.
- **Don't create a separate ExpenseCategoriesModule:** Expense categories are a simple name-only catalog entity. Add to existing CatalogsModule via the dimension map -- exact same pattern as supply_types.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Catalog CRUD for categories | New controller/service/module | CatalogsModule dimension map | Already handles 6 dimensions with FK-safe delete |
| Form validation | Manual validation | react-hook-form + zod | Already used everywhere |
| Toast notifications | Custom notification system | sonner (already wired) | Consistent UX |
| Collapsible groups | Custom accordion | Shadcn/ui Collapsible | Already used in SupplyTypeGroup |
| Confirmation dialogs | Custom modal | Shadcn/ui AlertDialog | Already used for destructive actions |

## Common Pitfalls

### Pitfall 1: TypeORM decimal returns string
**What goes wrong:** Amount shows as "45200.00" string instead of formatted number
**Why it happens:** TypeORM decimal columns return strings in JavaScript (same as SupplyPriceHistory.price)
**How to avoid:** Always `parseFloat(expense.amount)` on the frontend. Document in types.ts.
**Warning signs:** Amounts displaying with quotes or not formatting correctly

### Pitfall 2: Date timezone shift
**What goes wrong:** Expense entered for "2026-03-06" displays as "2026-03-05" due to UTC conversion
**Why it happens:** JavaScript Date objects apply timezone offset. If using `new Date("2026-03-06")` it creates midnight UTC which in UTC-3 (Argentina) is the previous day.
**How to avoid:** Use PostgreSQL `DATE` type (not timestamptz). On the frontend, treat the date string as-is ("2026-03-06") for display and grouping -- don't convert through `new Date()` for display purposes. Parse only when needed for comparison.
**Warning signs:** Expenses appearing in the wrong month group

### Pitfall 3: Category FK constraint on delete
**What goes wrong:** Category delete fails silently or shows generic error
**Why it happens:** FK constraint (23503) when expenses reference the category
**How to avoid:** CatalogsService already catches error code 23503 and throws ConflictException with message "No se puede eliminar: este item esta siendo usado por otros registros." The expense table FK to expense_categories will trigger this automatically.
**Warning signs:** None expected -- pattern already proven with supply_types

### Pitfall 4: Filter state not syncing with summary bar
**What goes wrong:** Summary bar shows grand total for ALL expenses while filters only show a subset
**Why it happens:** Summary computed from a different data source than the filtered list
**How to avoid:** Derive summary values (total, count, period) from the same filtered array that renders the month groups. Use `useMemo` to compute both from the same source.
**Warning signs:** Summary numbers don't match visible expenses

### Pitfall 5: Migration timestamp collision
**What goes wrong:** Migration doesn't run because timestamp conflicts with existing migration
**Why it happens:** Manually chosen timestamps that overlap
**How to avoid:** Use a timestamp higher than the last migration (1772500300000). Recommendation: use 1772600000000 for expense_categories table + seed, 1772600100000 for expenses table.

### Pitfall 6: Default date filter showing no data
**What goes wrong:** User opens expenses page and sees nothing because no expenses exist for current month
**Why it happens:** Default filter is current month, but seed data or first expenses may be from a different month
**How to avoid:** Show an appropriate empty state ("No hay gastos en este periodo. Usa el selector de fechas para ver otros meses.") and make the default date range the current month with clear visual indication of the active filter.

## Code Examples

### Backend: Expense Entity (complete)
```typescript
// Source: adapted from supply.entity.ts and supply-price-history.entity.ts patterns
import { Column, Entity, JoinColumn, ManyToOne } from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';
import { ExpenseCategory } from '../../catalogs/entities/expense-category.entity';

@Entity('expenses')
export class Expense extends BaseEntity {
  @Column({ type: 'decimal', precision: 12, scale: 2, nullable: false })
  amount!: string;

  @Column({ type: 'varchar', length: 500, nullable: false })
  concept!: string;

  @Column({ type: 'date', nullable: false })
  date!: string;

  @ManyToOne(() => ExpenseCategory, { eager: true, nullable: false })
  @JoinColumn({ name: 'category_id' })
  category!: ExpenseCategory;
}
```

### Backend: CreateExpenseDto
```typescript
import { IsDateString, IsNotEmpty, IsNumber, IsString, IsUUID, Min } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateExpenseDto {
  @ApiProperty({ example: 15000.50 })
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0.01)
  amount!: number;

  @ApiProperty({ example: 'Compra de cuero vacuno' })
  @IsString()
  @IsNotEmpty()
  concept!: string;

  @ApiProperty({ example: '2026-03-06' })
  @IsDateString()
  date!: string;

  @ApiProperty({ example: 'uuid-of-category' })
  @IsUUID()
  categoryId!: string;
}
```

### Backend: Adding expense-categories to CatalogsService
```typescript
// In catalogs.service.ts -- add to VALID_DIMENSIONS:
const VALID_DIMENSIONS = [
  'product-types',
  'product-names',
  'product-finishes',
  'product-colors',
  'product-sizes',
  'supply-types',
  'expense-categories', // NEW
] as const;

// In constructor -- add to dimensionMap:
this.dimensionMap = {
  // ... existing dimensions ...
  'expense-categories': this.expenseCategoryRepo as unknown as Repository<CatalogEntity>,
};

// DIMENSIONS_WITH_SKU stays unchanged (expense-categories don't have skuCode)
```

### Frontend: ExpenseFormDialog zod schema
```typescript
const expenseSchema = z.object({
  amount: z.string().min(1, 'El monto es obligatorio').refine(
    (val) => !isNaN(parseFloat(val)) && parseFloat(val) > 0,
    'El monto debe ser mayor a 0'
  ),
  concept: z.string().min(1, 'El concepto es obligatorio').max(500),
  date: z.string().min(1, 'La fecha es obligatoria'),
  categoryId: z.string().uuid('Selecciona una categoria'),
});
```

### Migration: Create expense_categories + seed
```typescript
public async up(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`
    CREATE TABLE "expense_categories" (
      "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
      "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
      "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
      "name" character varying(100) NOT NULL,
      CONSTRAINT "UQ_expense_categories_name" UNIQUE ("name"),
      CONSTRAINT "PK_expense_categories_id" PRIMARY KEY ("id")
    )
  `);

  // Seed 6 initial categories
  await queryRunner.query(`
    INSERT INTO "expense_categories" ("name") VALUES
    ('Materia prima'), ('Packaging'), ('Envio'),
    ('Herramientas'), ('Servicios'), ('Otros')
  `);

  await queryRunner.query(`
    CREATE TABLE "expenses" (
      "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
      "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
      "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
      "amount" decimal(12,2) NOT NULL,
      "concept" character varying(500) NOT NULL,
      "date" date NOT NULL,
      "category_id" uuid NOT NULL,
      CONSTRAINT "PK_expenses_id" PRIMARY KEY ("id"),
      CONSTRAINT "FK_expenses_category" FOREIGN KEY ("category_id") REFERENCES "expense_categories"("id")
    )
  `);

  -- Index on date for filter performance
  await queryRunner.query(`
    CREATE INDEX "IDX_expenses_date" ON "expenses" ("date" DESC)
  `);
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hardcoded enum for categories | DB catalog entity | User decision (Phase 7 context) | Categories editable without code changes |
| Soft delete for expenses | Hard delete | User decision (Phase 7 context) | Simpler -- no is_active filter needed |

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (backend e2e via NestJS) |
| Config file | nemea-back/test/jest-e2e.json |
| Quick run command | `cd nemea-back && npm run test` |
| Full suite command | `cd nemea-back && npm run test:e2e` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| EXPN-01 | Admin creates expense with amount, concept, date, category | e2e | `cd nemea-back && npx jest --config test/jest-e2e.json test/expenses.e2e-spec.ts` | Wave 0 |
| EXPN-02 | GET /expenses with category/date filters returns filtered results | e2e | same file | Wave 0 |
| EXPN-03 | Expense categories CRUD via catalogs dimension + seed | e2e | `cd nemea-back && npx jest --config test/jest-e2e.json test/catalogs.e2e-spec.ts` | Existing (extend) |

### Sampling Rate
- **Per task commit:** `cd nemea-back && npm run lint && npm run build`
- **Per wave merge:** `cd nemea-back && npm run test:e2e`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `nemea-back/test/expenses.e2e-spec.ts` -- covers EXPN-01, EXPN-02
- [ ] Extend existing catalogs e2e test with expense-categories dimension -- covers EXPN-03

## Open Questions

1. **Date range filter default behavior**
   - What we know: Default view is current month per user decision
   - What's unclear: Should the backend default to current month if no dateFrom/dateTo provided, or should the frontend set default query params?
   - Recommendation: Frontend sets default dateFrom/dateTo to current month boundaries. Backend returns all if no filter provided (simpler, more flexible).

2. **Summary bar scope**
   - What we know: Summary shows grand total + count + filtered period
   - What's unclear: "Grand total" means total of filtered results (shown expenses) or total of ALL expenses regardless of filter?
   - Recommendation: Total of filtered results. Label it clearly ("Total del periodo seleccionado"). This matches the CONTEXT.md note "updates dynamically with filters."

## Sources

### Primary (HIGH confidence)
- Project codebase -- all patterns verified by reading existing code:
  - `nemea-back/src/catalogs/` -- CatalogsService dimension map pattern
  - `nemea-back/src/supplies/` -- Entity with decimal amount, controller with QueryBuilder filtering
  - `nemea-front/src/components/supplies/SupplyTypeGroup.tsx` -- Collapsible grouped table pattern
  - `nemea-front/src/components/supplies/SupplyFormDialog.tsx` -- react-hook-form + zod dialog pattern
  - `nemea-front/src/components/catalogs/CatalogTabContent.tsx` -- Inline CRUD tab pattern
  - `nemea-front/src/components/layout/AppSidebar.tsx` -- Sidebar structure
  - `nemea-back/src/common/entities/base.entity.ts` -- BaseEntity with UUID

### Secondary (MEDIUM confidence)
- TypeORM decimal behavior (returns string) -- verified in existing SupplyPriceHistory.price pattern and supply types.ts `price: string`

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- zero new dependencies, all patterns established
- Architecture: HIGH -- direct adaptation of existing 6-phase patterns
- Pitfalls: HIGH -- most are already solved in existing code (decimal handling, FK constraint)

**Research date:** 2026-03-06
**Valid until:** 2026-04-06 (stable -- no external dependencies to change)
