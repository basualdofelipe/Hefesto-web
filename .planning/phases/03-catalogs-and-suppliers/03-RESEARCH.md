# Phase 3: Catalogs and Suppliers - Research

**Researched:** 2026-03-01
**Domain:** CRUD catalog entities + suppliers + UUID migration + sidebar navigation (NestJS + TypeORM + Next.js + Shadcn/ui)
**Confidence:** HIGH

## Summary

Phase 3 introduces the first real business entities beyond auth: six catalog dimension tables (product_type, product_name, product_finish, product_color, product_size, supply_type), a suppliers table with soft delete, and the frontend pages to manage them. This phase also carries a project-wide architectural change: migrating from integer SERIAL to UUID primary keys for all entities.

The backend work follows established NestJS module patterns already proven in phases 1-2. The six catalog entities are structurally identical (name-only tables extending BaseEntity), which means a single generic pattern can handle all six with minimal code duplication. The suppliers entity is slightly richer (multiple fields + soft delete via `is_active` with partial unique index).

The frontend introduces the first real application layout: a collapsible sidebar navigation replacing the current header-only layout. Shadcn/ui provides an official Sidebar component that handles collapsible states, mobile responsiveness, and icon-only mode out of the box. The catalog page uses a single Tabs component with six identical tab panels (inline CRUD lists), while suppliers get a table page + separate form page.

**Primary recommendation:** Implement backend entities and migrations first (including UUID migration), then build frontend pages. The UUID migration is the riskiest part -- it must alter the existing `users` table PK from integer to UUID, which requires dropping and recreating FK constraints in a single migration.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **Catalog UI layout:** Single page at `/catalogos` with 6 tabs: Tipos, Nombres, Terminaciones, Colores, Talles, Tipos de Insumo. All 6 catalogs are simple name-only tables presented identically within their tab. Reuse existing Shadcn/ui Tabs component.
- **Catalog editing:** Inline editing: click the item name to edit in-place, shows input + confirm/cancel. "Agregar" button shows an inline input at the top of the list for new items. No modal or separate form needed.
- **Catalog deletion:** Block deletion when the item is referenced by a product or supply. Show explanatory message: "No se puede eliminar: usado por X productos". Admin must remove FK reference first.
- **Catalog read-only access:** USER role can see the catalog page with all tabs and lists. Edit, delete, and add buttons are hidden for USER role (backend also enforces via @Roles).
- **Supplier list:** Table format with columns: Nombre, Email, Telefono, WhatsApp, Estado. Simple search box that filters by supplier name. Inactive suppliers remain visible but dimmed.
- **Supplier form:** Separate page: `/proveedores/nuevo` for create, `/proveedores/:id/editar` for edit. Full form with all fields: nombre (required), direccion, email, telefono, whatsapp, descripcion. Back button + Cancelar/Guardar buttons.
- **Supplier deactivation:** Toggle switch or status badge directly in the table row. Click to toggle with confirmation toast. Inactive suppliers stay visible but dimmed. Can reactivate.
- **Navigation:** Collapsible sidebar on the left with icons + labels. Logo moves to sidebar top. Toggle button to collapse to icons-only mode. Only show sections that are built. Phase 3 items: Inicio, Catalogos, Proveedores. Mobile: hamburger icon opens sidebar as overlay/drawer.
- **ID strategy (project-wide):** Migrate from integer SERIAL to UUID for all entities. BaseEntity.id changes to `@PrimaryGeneratedColumn('uuid')` (string type). Existing users table gets a migration to alter id from integer to UUID. All new entities born with UUID PKs. All FKs reference UUID columns.
- **Seed data:** Seed all 6 catalog dimensions with real business data from `Costos Nemea.xlsx`. Also seed supplier data. Idempotent seed: INSERT ... ON CONFLICT DO NOTHING. Seed runs as a TypeORM migration.

### Claude's Discretion
- Exact Tailwind styling, spacing, and color palette for active/inactive states
- Loading and error states
- Toast notification messages and duration
- Sidebar animation and collapse behavior
- Search debounce timing
- Table sorting (alphabetical default is fine)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| CATL-01 | Admin can CRUD product types (ej: Billetera, Cinturon) | Generic catalog entity pattern (name-only + BaseEntity), CatalogsModule with shared service, `@Roles(Role.ADMIN)` on write endpoints |
| CATL-02 | Admin can CRUD product names (ej: Hefesto, Ares) | Same generic catalog entity pattern as CATL-01 |
| CATL-03 | Admin can CRUD product finishes (ej: Lisa, Grabada) | Same generic catalog entity pattern as CATL-01 |
| CATL-04 | Admin can CRUD product colors (ej: Marron, Negro) | Same generic catalog entity pattern as CATL-01 |
| CATL-05 | Admin can CRUD product sizes (ej: Chico, Grande) | Same generic catalog entity pattern as CATL-01 |
| CATL-06 | Admin can CRUD supply types (ej: Cuero, Herraje, Packaging) | Same generic catalog entity pattern, registered in separate SuppliersModule or shared CatalogsModule |
| SUPP-01 | Admin can CRUD suppliers with name, address, email, phone, whatsapp, description | Supplier entity with multiple columns, SuppliersModule with standard NestJS CRUD, react-hook-form + zod for frontend form |
| SUPP-02 | Admin can deactivate/reactivate suppliers (soft delete) | `is_active` boolean column, partial unique index on name WHERE is_active = true, toggle endpoint PATCH /suppliers/:id/toggle-status |
</phase_requirements>

## Standard Stack

### Core (already installed)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | ^11.0.1 | Backend framework | Already installed, established patterns from phases 1-2 |
| TypeORM | ^0.3.28 | ORM + migrations | Already installed, data-source.ts is single source of truth |
| class-validator | ^0.14.4 | DTO validation | Already installed, used in CreateUserDto pattern |
| class-transformer | ^0.5.1 | DTO transformation | Already installed, needed for ValidationPipe transform |
| @nestjs/swagger | ^11.2.6 | API docs | Already installed, Swagger decorators on all controllers |
| Next.js | 16.1.6 | Frontend framework | Already installed, App Router |
| Shadcn/ui | new-york style | UI components | Already initialized (components.json), Tabs already installed |
| react-hook-form | ^7.71.2 | Form state management | Already installed, intended for supplier form |
| zod | ^4.3.6 | Schema validation | Already installed, for form validation with @hookform/resolvers |
| lucide-react | ^0.575.0 | Icons | Already installed, used in Header.tsx |
| sonner | ^2.0.7 | Toast notifications | Already installed, Toaster wired in layout.tsx |

### New Shadcn/ui Components to Install

| Component | Purpose | Install Command |
|-----------|---------|-----------------|
| sidebar | Collapsible sidebar navigation | `npx shadcn@latest add sidebar` |
| table | Supplier list table | `npx shadcn@latest add table` |
| switch | Supplier active/inactive toggle | `npx shadcn@latest add switch` |
| badge | Status badge (active/inactive) | `npx shadcn@latest add badge` |
| tooltip | Sidebar icon-only mode tooltips | `npx shadcn@latest add tooltip` |
| sheet | Mobile sidebar overlay (dependency of sidebar) | Auto-installed with sidebar |
| dialog | Deletion confirmation | `npx shadcn@latest add dialog` |

**Note:** The `sidebar` component from Shadcn/ui auto-installs its dependencies (sheet, tooltip, etc.). A single `npx shadcn@latest add sidebar` may pull in sheet and tooltip automatically. Verify after install.

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Shadcn Sidebar | Custom sidebar div | Shadcn handles collapse, mobile sheet, keyboard shortcut (Cmd+B) for free. Custom means reimplementing all of this. |
| Inline editing | Modal dialogs | User decided inline -- simpler UX for single-field entities. Modals would be overkill for a name field. |
| is_active boolean | @DeleteDateColumn (deleted_at) | User decided is_active in CONTEXT.md for suppliers. Partial unique index handles the soft-delete + unique constraint issue. Simpler mental model than deleted_at. |
| 6 separate controllers | 1 generic CatalogsController with dimension parameter | Generic controller reduces code duplication for 6 identical CRUD operations. But 6 separate controllers is also fine at this scale. Research recommends generic approach. |

**Installation:**
```bash
# Frontend -- new Shadcn components
cd nemea-front
npx shadcn@latest add sidebar table switch badge tooltip dialog

# No new npm packages needed -- all dependencies already installed
```

## Architecture Patterns

### Recommended Backend Structure
```
nemea-back/src/
├── catalogs/                       # CatalogsModule -- all 6 catalog dimensions
│   ├── catalogs.module.ts
│   ├── catalogs.controller.ts      # Generic CRUD for all 6 dimensions
│   ├── catalogs.service.ts         # Generic TypeORM operations
│   ├── entities/
│   │   ├── product-type.entity.ts
│   │   ├── product-name.entity.ts
│   │   ├── product-finish.entity.ts
│   │   ├── product-color.entity.ts
│   │   ├── product-size.entity.ts
│   │   └── supply-type.entity.ts
│   └── dto/
│       ├── create-catalog-item.dto.ts   # Shared: { name: string }
│       └── update-catalog-item.dto.ts   # Shared: { name: string }
│
├── suppliers/                      # SuppliersModule
│   ├── suppliers.module.ts
│   ├── suppliers.controller.ts     # CRUD + toggle-status
│   ├── suppliers.service.ts
│   ├── entities/
│   │   └── supplier.entity.ts
│   └── dto/
│       ├── create-supplier.dto.ts
│       └── update-supplier.dto.ts
│
├── common/
│   └── entities/
│       └── base.entity.ts          # Updated: UUID PK
```

### Recommended Frontend Structure
```
nemea-front/src/
├── app/
│   ├── layout.tsx                  # Updated: SidebarProvider + Sidebar replaces Header-only layout
│   ├── page.tsx                    # Inicio (dashboard placeholder)
│   ├── catalogos/
│   │   └── page.tsx                # Tabs with 6 catalog dimension lists
│   └── proveedores/
│       ├── page.tsx                # Supplier table with search
│       ├── nuevo/
│       │   └── page.tsx            # Create supplier form
│       └── [id]/
│           └── editar/
│               └── page.tsx        # Edit supplier form
│
├── components/
│   ├── layout/
│   │   ├── AppSidebar.tsx          # Sidebar component (replaces Header as primary nav)
│   │   ├── Header.tsx              # Simplified: SidebarTrigger + user menu (no more logo/nav)
│   │   └── SidebarLayout.tsx       # Wrapper: SidebarProvider + Sidebar + main content area
│   ├── catalogs/
│   │   ├── CatalogTabContent.tsx   # Reusable: list of items with inline CRUD
│   │   └── CatalogItemRow.tsx      # Single row: display mode + edit mode
│   └── suppliers/
│       ├── SupplierTable.tsx        # Table with search + status toggle
│       └── SupplierForm.tsx         # Form component (shared by create + edit pages)
│
├── hooks/
│   ├── useIsAdmin.ts               # Existing
│   └── useCatalog.ts               # Optional: fetch + mutate catalog items
│
├── lib/
│   ├── api.ts                      # Existing server-side fetch wrapper
│   └── api-client.ts               # New: client-side fetch wrapper for mutations
```

### Pattern 1: Generic Catalog CRUD (Backend)

**What:** All 6 catalog entities are structurally identical (UUID id, name, timestamps). A single service class with a generic entity parameter handles all CRUD operations, with a controller that routes by dimension name.

**When to use:** When multiple entities share the exact same structure and operations.

**Example:**
```typescript
// catalogs.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, ObjectType } from 'typeorm';

@Injectable()
export class CatalogsService {
  constructor(
    @InjectRepository(ProductType) private readonly productTypeRepo: Repository<ProductType>,
    @InjectRepository(ProductName) private readonly productNameRepo: Repository<ProductName>,
    @InjectRepository(ProductFinish) private readonly productFinishRepo: Repository<ProductFinish>,
    @InjectRepository(ProductColor) private readonly productColorRepo: Repository<ProductColor>,
    @InjectRepository(ProductSize) private readonly productSizeRepo: Repository<ProductSize>,
    @InjectRepository(SupplyType) private readonly supplyTypeRepo: Repository<SupplyType>,
  ) {}

  private getRepository(dimension: string): Repository<CatalogEntity> {
    const repos: Record<string, Repository<CatalogEntity>> = {
      'product-types': this.productTypeRepo,
      'product-names': this.productNameRepo,
      'product-finishes': this.productFinishRepo,
      'product-colors': this.productColorRepo,
      'product-sizes': this.productSizeRepo,
      'supply-types': this.supplyTypeRepo,
    };
    const repo = repos[dimension];
    if (!repo) throw new NotFoundException(`Dimension "${dimension}" no encontrada`);
    return repo;
  }

  async findAll(dimension: string): Promise<CatalogEntity[]> {
    return this.getRepository(dimension).find({ order: { name: 'ASC' } });
  }

  async create(dimension: string, dto: CreateCatalogItemDto): Promise<CatalogEntity> {
    const repo = this.getRepository(dimension);
    const item = repo.create(dto);
    return repo.save(item);
  }
  // ... update, remove (with FK check)
}
```

```typescript
// catalogs.controller.ts
@ApiTags('catalogs')
@ApiBearerAuth()
@Controller('catalogs')
export class CatalogsController {
  constructor(private readonly catalogsService: CatalogsService) {}

  @Get(':dimension')
  @ApiOperation({ summary: 'List all items in a catalog dimension' })
  async findAll(@Param('dimension') dimension: string): Promise<CatalogEntity[]> {
    return this.catalogsService.findAll(dimension);
  }

  @Post(':dimension')
  @Roles(Role.ADMIN)
  @ApiOperation({ summary: 'Create a catalog item (ADMIN only)' })
  async create(
    @Param('dimension') dimension: string,
    @Body() dto: CreateCatalogItemDto,
  ): Promise<CatalogEntity> {
    return this.catalogsService.create(dimension, dto);
  }

  // PUT :dimension/:id, DELETE :dimension/:id
}
```

### Pattern 2: UUID Migration for Existing Table

**What:** The `users` table currently has `id SERIAL` (integer). This migration must: (1) drop any FKs referencing users.id, (2) drop the PK constraint, (3) alter the column type to UUID using `gen_random_uuid()` for default, (4) cast existing integer IDs to UUID, (5) recreate PK and FKs.

**When to use:** One-time migration when changing PK strategy early in project lifecycle with minimal data.

**Critical detail:** PostgreSQL 13+ has `gen_random_uuid()` built-in (no extension needed). TypeORM's `@PrimaryGeneratedColumn('uuid')` generates `uuid_generate_v4()` which requires the `uuid-ossp` extension. The migration should install the extension OR use `gen_random_uuid()` directly.

**Example:**
```typescript
// Migration: AlterUsersIdToUuid
public async up(queryRunner: QueryRunner): Promise<void> {
  // Enable uuid-ossp extension (safe to run multiple times)
  await queryRunner.query(`CREATE EXTENSION IF NOT EXISTS "uuid-ossp"`);

  // Drop the existing PK constraint
  await queryRunner.query(`ALTER TABLE "users" DROP CONSTRAINT "PK_a3ffb1c0c8416b9fc6f907b7433"`);

  // Add a temporary UUID column
  await queryRunner.query(`ALTER TABLE "users" ADD COLUMN "new_id" uuid NOT NULL DEFAULT uuid_generate_v4()`);

  // Drop old id column and rename new one
  await queryRunner.query(`ALTER TABLE "users" DROP COLUMN "id"`);
  await queryRunner.query(`ALTER TABLE "users" RENAME COLUMN "new_id" TO "id"`);

  // Recreate PK
  await queryRunner.query(`ALTER TABLE "users" ADD CONSTRAINT "PK_users_id" PRIMARY KEY ("id")`);
}
```

**Note:** Since there are no FK references to users.id from other tables yet (only 1 table exists), this migration is straightforward. The seed admin user will get a new UUID id.

### Pattern 3: Inline Editing Component (Frontend)

**What:** Each catalog item row has two states: display mode (shows name + edit/delete icons) and edit mode (shows input + confirm/cancel). State is managed locally per row.

**When to use:** Single-field entities where a full form would be overkill.

**Example:**
```typescript
// CatalogItemRow.tsx
'use client';

interface CatalogItemRowProps {
  item: { id: string; name: string };
  isAdmin: boolean;
  onUpdate: (id: string, name: string) => Promise<void>;
  onDelete: (id: string) => Promise<void>;
}

export function CatalogItemRow({ item, isAdmin, onUpdate, onDelete }: CatalogItemRowProps): ReactElement {
  const [isEditing, setIsEditing] = useState(false);
  const [editValue, setEditValue] = useState(item.name);

  async function handleSave(): Promise<void> {
    await onUpdate(item.id, editValue);
    setIsEditing(false);
  }

  if (isEditing && isAdmin) {
    return (
      <div className="flex items-center gap-2">
        <Input value={editValue} onChange={(e) => setEditValue(e.target.value)} />
        <Button size="icon" variant="ghost" onClick={handleSave}><Check /></Button>
        <Button size="icon" variant="ghost" onClick={() => setIsEditing(false)}><X /></Button>
      </div>
    );
  }

  return (
    <div className="flex items-center justify-between">
      <span>{item.name}</span>
      {isAdmin && (
        <div className="flex gap-1">
          <Button size="icon" variant="ghost" onClick={() => setIsEditing(true)}><Pencil /></Button>
          <Button size="icon" variant="ghost" onClick={() => onDelete(item.id)}><Trash2 /></Button>
        </div>
      )}
    </div>
  );
}
```

### Pattern 4: Shadcn Sidebar Layout

**What:** Shadcn/ui's Sidebar component provides a SidebarProvider context, collapsible sidebar with icon-only mode, and automatic mobile sheet/drawer behavior.

**When to use:** Any app that needs a persistent navigation sidebar with collapse support.

**Example layout integration:**
```typescript
// app/layout.tsx (updated)
export default function RootLayout({ children }: { children: ReactNode }): ReactElement {
  return (
    <html lang="es" suppressHydrationWarning>
      <body>
        <ThemeProvider>
          <SessionProvider>
            <SidebarProvider>
              <AppSidebar />
              <main className="flex-1">
                <Header /> {/* Now just SidebarTrigger + user menu */}
                {children}
              </main>
            </SidebarProvider>
            <Toaster />
          </SessionProvider>
        </ThemeProvider>
      </body>
    </html>
  );
}
```

```typescript
// components/layout/AppSidebar.tsx
import { Home, BookOpen, Truck } from 'lucide-react';
import {
  Sidebar,
  SidebarContent,
  SidebarGroup,
  SidebarGroupContent,
  SidebarHeader,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
} from '@/components/ui/sidebar';

const navItems = [
  { title: 'Inicio', href: '/', icon: Home },
  { title: 'Catalogos', href: '/catalogos', icon: BookOpen },
  { title: 'Proveedores', href: '/proveedores', icon: Truck },
];

export function AppSidebar(): ReactElement {
  return (
    <Sidebar collapsible="icon">
      <SidebarHeader>
        <span className="engraving-title text-primary text-xl tracking-widest">NEMEA</span>
      </SidebarHeader>
      <SidebarContent>
        <SidebarGroup>
          <SidebarGroupContent>
            <SidebarMenu>
              {navItems.map((item) => (
                <SidebarMenuItem key={item.title}>
                  <SidebarMenuButton asChild>
                    <Link href={item.href}>
                      <item.icon />
                      <span>{item.title}</span>
                    </Link>
                  </SidebarMenuButton>
                </SidebarMenuItem>
              ))}
            </SidebarMenu>
          </SidebarGroupContent>
        </SidebarGroup>
      </SidebarContent>
    </Sidebar>
  );
}
```

### Pattern 5: Client-Side API Calls for Mutations

**What:** The existing `api.ts` uses `auth()` which is server-side only (it reads the session from cookies on the server). For client components that perform mutations (inline edit, toggle status), a client-side fetch wrapper is needed that reads the session token via `useSession()`.

**When to use:** Any client component that needs to POST/PUT/PATCH/DELETE to the backend.

**Example:**
```typescript
// lib/api-client.ts
'use client';

const API_URL = process.env.NEXT_PUBLIC_API_URL;

export async function apiClientFetch<T>(
  path: string,
  token: string,
  options: RequestInit = {},
): Promise<T> {
  const res = await fetch(`${API_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
      ...options.headers,
    },
  });

  if (!res.ok) {
    const error = await res.json().catch(() => ({ message: `Error ${res.status}` }));
    throw new Error(error.message ?? `API error: ${res.status}`);
  }

  return res.json() as Promise<T>;
}
```

### Anti-Patterns to Avoid

- **6 separate modules for 6 identical entities:** All catalog dimensions are structurally identical (name-only). Creating 6 separate NestJS modules would be unnecessary duplication. Use a single CatalogsModule with a generic service.
- **Server actions for inline editing:** Server actions (Next.js) work well for form submissions but are clunky for inline editing in client components. Use client-side fetch for mutations in interactive components.
- **Hardcoded sidebar items for future phases:** Only include sidebar items for pages that exist. Do not add disabled/grayed-out items for phases 4-7.
- **Exposing entity IDs as sequential integers:** The UUID migration exists specifically to avoid this. Never expose predictable IDs in URLs or API responses.
- **Using `synchronize: true` to apply new entity schemas:** Always generate migrations. Six new entity tables = one migration file.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Collapsible sidebar | Custom sidebar with useState + CSS transitions | Shadcn/ui Sidebar component | Handles collapse states (full, icon, offcanvas), mobile sheet, keyboard shortcut (Cmd+B), cookie persistence of state |
| Mobile sidebar drawer | Custom drawer with portal + backdrop | Shadcn Sidebar (mobile mode auto) | Sidebar automatically renders as Sheet on mobile viewports |
| UUID generation | Custom UUID function | PostgreSQL `uuid_generate_v4()` via `@PrimaryGeneratedColumn('uuid')` | TypeORM handles UUID generation at DB level |
| Form validation | Manual validation in handlers | react-hook-form + zod schemas + @hookform/resolvers | Already installed, handles validation, error messages, dirty tracking |
| Toast notifications | Custom notification system | sonner (already wired) | Already in layout.tsx, just call `toast()` or `toast.success()` |
| Debounced search | Custom setTimeout/clearTimeout | useDeferredValue (React 19) or simple debounce util | React 19's useDeferredValue handles search input debouncing natively without extra dependencies |

**Key insight:** This phase introduces no new npm dependencies. Every library needed is already installed. The work is entities, migrations, components, and wiring.

## Common Pitfalls

### Pitfall 1: UUID Migration Breaking Admin Seed User
**What goes wrong:** The existing CreateUserTable migration seeds an admin user with integer id. After the UUID migration, the `auth.ts` file on the frontend stores `token.userId = String(body.data.user.id)` as a string. If the UUID migration re-creates the user or changes the id, the existing JWT tokens become invalid because `userId` no longer matches.
**Why it happens:** The migration must handle existing data carefully -- it cannot just drop and recreate the table.
**How to avoid:** The UUID migration should convert existing ids in-place (add UUID column, copy data, drop old column). After migration, existing users get new UUID ids. Since JWT TTL is 7 days and tokens are re-issued on login, users will naturally get new tokens with the correct UUID. Force a re-login if needed by changing JWT_SECRET.
**Warning signs:** 401 errors after migration on existing sessions. User exists in DB but guard fails to find by id.

### Pitfall 2: TypeORM uuid_generate_v4 Requires Extension
**What goes wrong:** `@PrimaryGeneratedColumn('uuid')` generates SQL with `DEFAULT uuid_generate_v4()`. This function lives in the `uuid-ossp` PostgreSQL extension, which is NOT enabled by default.
**Why it happens:** TypeORM assumes the extension exists. PostgreSQL 13+ has `gen_random_uuid()` built-in, but TypeORM uses the older function.
**How to avoid:** Add `CREATE EXTENSION IF NOT EXISTS "uuid-ossp"` as the first statement in the UUID migration. This is idempotent (safe to run multiple times). Railway PostgreSQL supports this extension.
**Warning signs:** `ERROR: function uuid_generate_v4() does not exist` when running migrations.

### Pitfall 3: Catalog Deletion Fails Silently When Referenced
**What goes wrong:** An admin tries to delete a product_type that is referenced by products in phase 5. Without a FK check, the delete succeeds and orphans the product records. With a DB FK constraint, it throws a raw PostgreSQL error that leaks to the frontend.
**Why it happens:** FK constraints between catalogs and products don't exist yet (products table is phase 5). But the delete endpoint must be designed to handle this from the start.
**How to avoid:** Design the delete endpoint to check for FK references before deleting. In phase 3, there are no references yet, but the endpoint should query for references and return a 409 Conflict with a message like "No se puede eliminar: usado por X productos" when references exist. This check will become real when products are created in phase 5. For now, the check returns 0 references and the delete proceeds.
**Warning signs:** Delete button works in phase 3 but starts failing in phase 5 without clear error messages.

### Pitfall 4: Sidebar Layout Breaks Login Page
**What goes wrong:** The sidebar is added to `layout.tsx` (root layout), which wraps ALL pages including `/login` and `/acceso-denegado`. These pages should NOT show the sidebar.
**Why it happens:** Root layout applies to every route in Next.js App Router.
**How to avoid:** Use Next.js route groups: `(auth)` group for login/acceso-denegado (no sidebar), `(app)` group for authenticated pages (with sidebar). Or conditionally render the sidebar based on session state. The route group approach is cleaner.
**Warning signs:** Login page shows a broken sidebar with no user data.

### Pitfall 5: BaseEntity UUID Change Breaks UsersService.findById
**What goes wrong:** `UsersService.findById(id: number)` takes a number parameter. After UUID migration, ids are strings. All service methods that accept id parameters must change from `number` to `string`. The `ParseIntPipe` in `UsersController.remove(@Param('id', ParseIntPipe) id: number)` must change to `ParseUUIDPipe`.
**Why it happens:** The UUID migration changes the PK type but doesn't automatically update the TypeScript code that references it.
**How to avoid:** After updating BaseEntity, grep the entire codebase for `id: number` and `ParseIntPipe` references. Update all of them to `id: string` and `ParseUUIDPipe` in the same commit.
**Warning signs:** TypeScript errors on `findById(id)` calls after changing BaseEntity. Runtime 400 errors when passing UUID strings to endpoints expecting integers.

### Pitfall 6: Partial Unique Index Not Created for Supplier Name
**What goes wrong:** Supplier has `is_active` for soft delete and `name` should be unique among active suppliers. A standard `@Unique(['name'])` prevents creating a new supplier with the same name as a deactivated one.
**Why it happens:** Standard unique constraints don't filter by `is_active`.
**How to avoid:** Do NOT use `@Unique(['name'])` on the entity. Instead, create a partial unique index in the migration: `CREATE UNIQUE INDEX "idx_suppliers_name_active" ON "suppliers" ("name") WHERE "is_active" = true`. TypeORM's `@Index` decorator supports this: `@Index(['name'], { where: '"is_active" = true', unique: true })`.
**Warning signs:** `QueryFailedError: duplicate key value violates unique constraint` when creating a supplier with the same name as a deactivated one.

## Code Examples

### Catalog Entity (all 6 follow this pattern)

```typescript
// catalogs/entities/product-type.entity.ts
import { Column, Entity } from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';

@Entity('product_types')
export class ProductType extends BaseEntity {
  @Column({ type: 'varchar', length: 100, unique: true })
  name!: string;
}
```

### Supplier Entity

```typescript
// suppliers/entities/supplier.entity.ts
import { Column, Entity, Index } from 'typeorm';
import { BaseEntity } from '../../common/entities/base.entity';

@Entity('suppliers')
@Index(['name'], { where: '"is_active" = true', unique: true })
export class Supplier extends BaseEntity {
  @Column({ type: 'varchar', length: 255, nullable: false })
  name!: string;

  @Column({ type: 'varchar', length: 500, nullable: true })
  address!: string | null;

  @Column({ type: 'varchar', length: 255, nullable: true })
  email!: string | null;

  @Column({ type: 'varchar', length: 50, nullable: true })
  phone!: string | null;

  @Column({ type: 'varchar', length: 50, nullable: true })
  whatsapp!: string | null;

  @Column({ type: 'text', nullable: true })
  description!: string | null;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive!: boolean;
}
```

### Updated BaseEntity with UUID

```typescript
// common/entities/base.entity.ts
import {
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @CreateDateColumn({ name: 'created_at', type: 'timestamptz' })
  createdAt!: Date;

  @UpdateDateColumn({ name: 'updated_at', type: 'timestamptz' })
  updatedAt!: Date;
}
```

### Supplier DTO

```typescript
// suppliers/dto/create-supplier.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsNotEmpty, IsOptional, IsString, MaxLength } from 'class-validator';

export class CreateSupplierDto {
  @ApiProperty({ description: 'Nombre del proveedor', example: 'Curtiembre Central' })
  @IsString()
  @IsNotEmpty()
  @MaxLength(255)
  name!: string;

  @ApiProperty({ required: false, description: 'Direccion' })
  @IsString()
  @IsOptional()
  @MaxLength(500)
  address?: string;

  @ApiProperty({ required: false, description: 'Email del proveedor' })
  @IsEmail()
  @IsOptional()
  email?: string;

  @ApiProperty({ required: false, description: 'Telefono' })
  @IsString()
  @IsOptional()
  @MaxLength(50)
  phone?: string;

  @ApiProperty({ required: false, description: 'WhatsApp' })
  @IsString()
  @IsOptional()
  @MaxLength(50)
  whatsapp?: string;

  @ApiProperty({ required: false, description: 'Descripcion o notas sobre el proveedor' })
  @IsString()
  @IsOptional()
  description?: string;
}
```

### Seed Migration Pattern

```typescript
// Seed migration (runs after table creation migration)
public async up(queryRunner: QueryRunner): Promise<void> {
  // Product types
  await queryRunner.query(`
    INSERT INTO "product_types" ("name") VALUES
      ('Billetera'), ('Cinturon'), ('Deskpad'), ('Porta Notebook')
    ON CONFLICT ("name") DO NOTHING
  `);

  // Product names
  await queryRunner.query(`
    INSERT INTO "product_names" ("name") VALUES
      ('Hefesto'), ('Ares'), ('Hermes'), ('Apolo'), ('Poseidon')
    ON CONFLICT ("name") DO NOTHING
  `);

  // Product finishes
  await queryRunner.query(`
    INSERT INTO "product_finishes" ("name") VALUES
      ('Lisa'), ('Grabada'), ('Saffiano')
    ON CONFLICT ("name") DO NOTHING
  `);

  // Product colors
  await queryRunner.query(`
    INSERT INTO "product_colors" ("name") VALUES
      ('Marron'), ('Negro'), ('Suela'), ('Bordo'), ('Azul')
    ON CONFLICT ("name") DO NOTHING
  `);

  // Product sizes
  await queryRunner.query(`
    INSERT INTO "product_sizes" ("name") VALUES
      ('Unico'), ('Chico'), ('Mediano'), ('Grande')
    ON CONFLICT ("name") DO NOTHING
  `);

  // Supply types
  await queryRunner.query(`
    INSERT INTO "supply_types" ("name") VALUES
      ('Cuero'), ('Herraje'), ('Packaging'), ('Hilo'), ('Adhesivo')
    ON CONFLICT ("name") DO NOTHING
  `);

  // Suppliers (example -- extract real data from Excel)
  await queryRunner.query(`
    INSERT INTO "suppliers" ("name", "description") VALUES
      ('Curtiembre Central', 'Proveedor principal de cueros')
    ON CONFLICT DO NOTHING
  `);
}
```

**Note:** The exact seed values must be extracted from `Costos Nemea.xlsx` during implementation. The examples above are based on business logic documentation and Apps Script references. The planner should include a task to extract real data from the Excel file.

### Supplier Form with react-hook-form + zod

```typescript
// components/suppliers/SupplierForm.tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const supplierSchema = z.object({
  name: z.string().min(1, 'El nombre es obligatorio').max(255),
  address: z.string().max(500).optional().or(z.literal('')),
  email: z.string().email('Email invalido').optional().or(z.literal('')),
  phone: z.string().max(50).optional().or(z.literal('')),
  whatsapp: z.string().max(50).optional().or(z.literal('')),
  description: z.string().optional().or(z.literal('')),
});

type SupplierFormData = z.infer<typeof supplierSchema>;

interface SupplierFormProps {
  defaultValues?: SupplierFormData;
  onSubmit: (data: SupplierFormData) => Promise<void>;
  isLoading?: boolean;
}

export function SupplierForm({ defaultValues, onSubmit, isLoading }: SupplierFormProps): ReactElement {
  const form = useForm<SupplierFormData>({
    resolver: zodResolver(supplierSchema),
    defaultValues: defaultValues ?? { name: '', address: '', email: '', phone: '', whatsapp: '', description: '' },
  });

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields using Shadcn Input + Label */}
    </form>
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Integer SERIAL PKs | UUID PKs (`gen_random_uuid()`) | PostgreSQL 13+ (2020) | Non-sequential, non-guessable IDs. Standard for modern APIs. TypeORM supports via `@PrimaryGeneratedColumn('uuid')` |
| Custom sidebar components | Shadcn/ui Sidebar (composable, themeable) | Shadcn/ui 2024+ | Handles collapse, mobile, keyboard shortcuts, cookie persistence |
| Manual form state | react-hook-form + zod | Standard since ~2022 | Zero re-renders on input change, schema-based validation, type-safe |
| Server-side rendering only | RSC + client components hybrid | Next.js 13+ App Router | Catalog lists can be server-rendered (fast initial load), mutations use client components |
| `@DeleteDateColumn` for soft delete | Either approach valid | Ongoing | `is_active` chosen per user decision -- simpler mental model, explicit in queries |

**Deprecated/outdated:**
- `autoLoadEntities: true` in TypeORM NestJS config: This project uses shared `data-source.ts` with explicit entity paths instead (Pitfall 2 from project research).
- `uuid-ossp` extension vs `gen_random_uuid()`: Both work. TypeORM generates `uuid_generate_v4()` which needs the extension. Adding `CREATE EXTENSION IF NOT EXISTS "uuid-ossp"` is the simplest fix.
- `ParseIntPipe` for UUID params: Must switch to `ParseUUIDPipe` after UUID migration.

## Open Questions

1. **Exact seed data values from Excel**
   - What we know: The `Costos Nemea.xlsx` file contains the real business data (product types, names, finishes, colors, suppliers)
   - What's unclear: The exact values in each sheet (Excel is binary, not readable as text)
   - Recommendation: During implementation, open the Excel file manually or use a script to extract the seed values. The planner should include a task for this. Business logic docs mention: Billetera, Cinturon (types), Hefesto, Ares (names), Lisa, Grabada (finishes), Marron, Negro (colors), Cuero, Herraje, Packaging (supply types).

2. **Route groups for authenticated vs public pages**
   - What we know: Login and acceso-denegado pages should not show the sidebar. Authenticated pages should.
   - What's unclear: Whether to use Next.js route groups `(auth)/(app)` or conditional rendering in layout.
   - Recommendation: Use route groups. Create `app/(app)/layout.tsx` with sidebar, and `app/(auth)/layout.tsx` without. Move existing login and acceso-denegado into `(auth)` group, all other pages into `(app)` group. This is the standard Next.js App Router pattern.

3. **Client-side data fetching strategy**
   - What we know: `api.ts` is server-side only (uses `auth()` from `@/auth`). Client components need a different approach for mutations.
   - What's unclear: Whether to use a dedicated client-side fetch wrapper or React Server Actions for form submissions.
   - Recommendation: Use React Server Actions for the supplier form (full-page form with redirect) and a client-side fetch wrapper for inline catalog mutations (no page navigation needed). This gives the best of both worlds.

## Sources

### Primary (HIGH confidence)
- Existing codebase analysis: `base.entity.ts`, `user.entity.ts`, `users.controller.ts`, `users.service.ts`, `data-source.ts`, `app.module.ts`, `main.ts` -- patterns established in phases 1-2
- Existing CONTEXT.md: User decisions from discuss-phase session (2026-03-01)
- Project REQUIREMENTS.md: CATL-01 through CATL-06, SUPP-01, SUPP-02
- Project ARCHITECTURE.md: Module structure, recommended project layout
- Project PITFALLS.md: Soft delete + unique constraint (Pitfall 6), UUID migration considerations

### Secondary (MEDIUM confidence)
- [Shadcn/ui Sidebar documentation](https://ui.shadcn.com/docs/components/radix/sidebar) -- Component structure, installation, collapsible modes, mobile behavior
- [TypeORM issue #7549: Unique Index and soft delete](https://github.com/typeorm/typeorm/issues/7549) -- Partial unique index with `@Index` decorator WHERE clause
- [TypeORM issue #11571: uuid_generate_v4 function](https://github.com/typeorm/typeorm/issues/11571) -- Extension requirement for UUID generation
- [Wanago.io: NestJS soft deletes with PostgreSQL and TypeORM](https://wanago.io/2021/10/25/api-nestjs-soft-deletes-postgresql-typeorm/) -- is_active vs deleted_at patterns

### Tertiary (LOW confidence)
- Seed data values (Billetera, Hefesto, etc.) -- Inferred from Apps Script references and business logic documentation. Must be verified against actual Excel data during implementation.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All libraries already installed, no new dependencies. Patterns established in phases 1-2.
- Architecture: HIGH -- NestJS module pattern is well-established. Generic catalog service is a standard DRY approach. Shadcn Sidebar is official and well-documented.
- Pitfalls: HIGH -- UUID migration is the riskiest area but well-understood (only 1 existing table, no FKs). Soft delete + partial unique index is a documented TypeORM pattern.

**Research date:** 2026-03-01
**Valid until:** 2026-03-31 (stable domain, no fast-moving dependencies)
