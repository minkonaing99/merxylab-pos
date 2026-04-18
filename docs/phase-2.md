# Phase 2 — Stock & Products Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the global product catalog (CRUD + image upload) and per-store inventory system (stock levels + append-only movement ledger) with full frontend panels.

**Architecture:** Products live in a global catalog shared across all stores. `StoreProduct` bridges each product to a store with its own `stock_qty`. `StockMovement` is an append-only ledger — every stock change writes a row and atomically updates `StoreProduct.stock_qty` in the same Prisma transaction. Images are uploaded to S3-compatible storage and resized to three sizes via Sharp.

**Tech Stack:** NestJS modules (products, categories, inventory, uploads), Prisma transactions, Multer + Sharp + MinIO/S3, Next.js Stock panel, TanStack Table, React Hook Form + Zod, slide-over drawers

**Depends on:** Phase 1 complete (auth, PrismaModule, RedisModule, design system)

---

## File Map

```
apps/api/src/
├── prisma/schema.prisma                      # extend with product + inventory tables
├── uploads/
│   ├── uploads.module.ts
│   ├── uploads.controller.ts
│   └── uploads.service.ts                    # Sharp resize + S3 upload
├── products/
│   ├── products.module.ts
│   ├── products.controller.ts
│   ├── products.service.ts
│   ├── products.service.spec.ts
│   └── dto/
│       ├── create-product.dto.ts
│       └── update-product.dto.ts
├── categories/
│   ├── categories.module.ts
│   ├── categories.controller.ts
│   ├── categories.service.ts
│   └── dto/create-category.dto.ts
├── brands/
│   ├── brands.module.ts
│   ├── brands.controller.ts
│   └── brands.service.ts
├── units/
│   ├── units.module.ts
│   ├── units.controller.ts
│   └── units.service.ts
└── inventory/
    ├── inventory.module.ts
    ├── inventory.controller.ts
    ├── inventory.service.ts
    └── inventory.service.spec.ts

apps/web/src/
└── app/(stock)/
    ├── stock/products/
    │   ├── page.tsx                           # product list (TanStack Table)
    │   └── _components/
    │       ├── ProductDrawer.tsx              # add/edit slide-over
    │       └── ImageUploader.tsx
    ├── stock/categories/page.tsx
    ├── stock/overview/page.tsx
    └── stock/movements/page.tsx

packages/types/src/
├── product.types.ts
└── inventory.types.ts
```

---

## Task 1: Extend Prisma schema — product + inventory tables

**Files:**
- Modify: `apps/api/prisma/schema.prisma`

- [ ] **Step 1: Add enums and models to schema.prisma**

Append to the existing `schema.prisma` (after the `UserStoreRole` model):

```prisma
enum StockMovementType {
  SALE
  RECEIVE
  ADJUST
  TRANSFER_OUT
  TRANSFER_IN
}

model Product {
  id             String   @id @default(cuid())
  name           String
  sku            String   @unique
  barcode        String?  @unique
  categoryId     String?  @map("category_id")
  brandId        String?  @map("brand_id")
  unitId         String?  @map("unit_id")
  costPrice      Int      @default(0) @map("cost_price")
  sellPrice      Int      @default(0) @map("sell_price")
  imageUrls      String[] @map("image_urls")
  mainImageIndex Int      @default(0) @map("main_image_index")
  isActive       Boolean  @default(true) @map("is_active")
  isTrackable    Boolean  @default(true) @map("is_trackable")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  category   Category?      @relation(fields: [categoryId], references: [id])
  brand      Brand?         @relation(fields: [brandId], references: [id])
  unit       Unit?          @relation(fields: [unitId], references: [id])
  storeStock StoreProduct[]
  movements  StockMovement[]

  @@map("products")
}

model Category {
  id        String     @id @default(cuid())
  name      String
  slug      String     @unique
  parentId  String?    @map("parent_id")
  sortOrder Int        @default(0) @map("sort_order")

  parent   Category?  @relation("CategoryTree", fields: [parentId], references: [id])
  children Category[] @relation("CategoryTree")
  products Product[]

  @@map("categories")
}

model Brand {
  id       String    @id @default(cuid())
  name     String    @unique
  products Product[]

  @@map("brands")
}

model Unit {
  id           String    @id @default(cuid())
  name         String    @unique
  abbreviation String
  products     Product[]

  @@map("units")
}

model StoreProduct {
  id          String @id @default(cuid())
  storeId     String @map("store_id")
  productId   String @map("product_id")
  stockQty    Int    @default(0) @map("stock_qty")
  minStockQty Int    @default(5) @map("min_stock_qty")
  maxStockQty Int?   @map("max_stock_qty")

  store   Store   @relation(fields: [storeId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([storeId, productId])
  @@map("store_products")
}

model StockMovement {
  id          String            @id @default(cuid())
  storeId     String            @map("store_id")
  productId   String            @map("product_id")
  type        StockMovementType
  qtyChange   Int               @map("qty_change")
  referenceId String?           @map("reference_id")
  note        String?
  createdById String            @map("created_by")
  createdAt   DateTime          @default(now()) @map("created_at")

  store     Store   @relation(fields: [storeId], references: [id])
  product   Product @relation(fields: [productId], references: [id])
  createdBy User    @relation(fields: [createdById], references: [id])

  @@map("stock_movements")
}
```

Also add back-relations to existing models:

In the `Store` model add:
```prisma
  storeProducts  StoreProduct[]
  stockMovements StockMovement[]
```

In the `User` model add:
```prisma
  stockMovements StockMovement[]
```

- [ ] **Step 2: Run migration**

```bash
cd apps/api && pnpm prisma migrate dev --name add-products-inventory
```

Expected: `✔ Generated Prisma Client`

- [ ] **Step 3: Add product/inventory types to packages/types**

Create `packages/types/src/product.types.ts`:

```typescript
export interface Product {
  id: string
  name: string
  sku: string
  barcode?: string | null
  categoryId?: string | null
  brandId?: string | null
  unitId?: string | null
  costPrice: number
  sellPrice: number
  imageUrls: string[]
  mainImageIndex: number
  isActive: boolean
  isTrackable: boolean
  createdAt: string
  updatedAt: string
  category?: { id: string; name: string } | null
  brand?: { id: string; name: string } | null
  unit?: { id: string; name: string; abbreviation: string } | null
}

export interface CreateProductDto {
  name: string
  sku?: string
  barcode?: string
  categoryId?: string
  brandId?: string
  unitId?: string
  costPrice: number
  sellPrice: number
  imageUrls?: string[]
  mainImageIndex?: number
  isActive?: boolean
  isTrackable?: boolean
}

export interface StoreProduct {
  id: string
  storeId: string
  productId: string
  stockQty: number
  minStockQty: number
  maxStockQty?: number | null
  product?: Product
}
```

Create `packages/types/src/inventory.types.ts`:

```typescript
import { StockMovementType } from './enums'

export interface StockMovement {
  id: string
  storeId: string
  productId: string
  type: StockMovementType
  qtyChange: number
  referenceId?: string | null
  note?: string | null
  createdById: string
  createdAt: string
  product?: { name: string; sku: string }
  createdBy?: { name: string }
}

export interface StockAdjustmentDto {
  productId: string
  newQty: number
  reason: string
}
```

Update `packages/types/src/index.ts` to export new files:

```typescript
export * from './enums'
export * from './auth.types'
export * from './api.types'
export * from './product.types'
export * from './inventory.types'
```

- [ ] **Step 4: Rebuild types package**

```bash
cd packages/types && pnpm build
```

- [ ] **Step 5: Commit**

```bash
cd ../..
git add apps/api/prisma packages/types
git commit -m "feat: extend Prisma schema with product and inventory models"
```

---

## Task 2: Uploads service (image resize + S3)

**Files:**
- Create: `apps/api/src/uploads/uploads.service.ts`
- Create: `apps/api/src/uploads/uploads.controller.ts`
- Create: `apps/api/src/uploads/uploads.module.ts`

- [ ] **Step 1: Install Sharp and S3 client**

```bash
cd apps/api && pnpm add sharp @aws-sdk/client-s3 @aws-sdk/lib-storage multer fastify-multer
pnpm add -D @types/multer
```

Add to `.env.example` and `.env`:
```env
S3_ENDPOINT=http://localhost:9000
S3_REGION=us-east-1
S3_BUCKET=merxylab-pos
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_PUBLIC_URL=http://localhost:9000/merxylab-pos
```

Start MinIO locally:
```bash
docker run -d --name pos-minio \
  -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  minio/minio server /data --console-address ":9001"
```

Create bucket `merxylab-pos` at http://localhost:9001 (minioadmin/minioadmin), set to public.

- [ ] **Step 2: Create apps/api/src/uploads/uploads.service.ts**

```typescript
import { Injectable, BadRequestException } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'
import * as sharp from 'sharp'
import { randomUUID } from 'crypto'

const SIZES = [
  { suffix: 'sm', width: 64 },
  { suffix: 'md', width: 256 },
  { suffix: 'lg', width: 800 },
]

const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp']
const MAX_BYTES = 2 * 1024 * 1024 // 2 MB

@Injectable()
export class UploadsService {
  private s3: S3Client
  private bucket: string
  private publicUrl: string

  constructor(private config: ConfigService) {
    this.bucket = config.get<string>('S3_BUCKET')!
    this.publicUrl = config.get<string>('S3_PUBLIC_URL')!
    this.s3 = new S3Client({
      endpoint: config.get<string>('S3_ENDPOINT'),
      region: config.get<string>('S3_REGION') ?? 'us-east-1',
      credentials: {
        accessKeyId: config.get<string>('S3_ACCESS_KEY')!,
        secretAccessKey: config.get<string>('S3_SECRET_KEY')!,
      },
      forcePathStyle: true,
    })
  }

  async uploadProductImage(
    buffer: Buffer,
    mimeType: string,
  ): Promise<string> {
    if (!ALLOWED_MIME.includes(mimeType)) {
      throw new BadRequestException('Only JPG, PNG, WEBP allowed')
    }
    if (buffer.length > MAX_BYTES) {
      throw new BadRequestException('Image must be under 2MB')
    }

    const id = randomUUID()
    const urls: string[] = []

    for (const size of SIZES) {
      const resized = await sharp(buffer)
        .resize(size.width, size.width, { fit: 'cover' })
        .webp({ quality: 85 })
        .toBuffer()

      const key = `products/${id}/${size.suffix}.webp`
      await this.s3.send(
        new PutObjectCommand({
          Bucket: this.bucket,
          Key: key,
          Body: resized,
          ContentType: 'image/webp',
        }),
      )
      urls.push(`${this.publicUrl}/${key}`)
    }

    // Return the lg URL as the stored key (frontend uses sm/md/lg suffix pattern)
    return `products/${id}`
  }

  getImageUrl(key: string, size: 'sm' | 'md' | 'lg' = 'md'): string {
    return `${this.publicUrl}/${key}/${size}.webp`
  }
}
```

- [ ] **Step 3: Create apps/api/src/uploads/uploads.controller.ts**

```typescript
import {
  Controller,
  Post,
  UseGuards,
  UseInterceptors,
  UploadedFile,
  BadRequestException,
} from '@nestjs/common'
import { FileInterceptor } from '@nestjs/platform-express'
import { UploadsService } from './uploads.service'
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard'
import { Roles } from '../auth/decorators/roles.decorator'
import { RolesGuard } from '../auth/guards/roles.guard'
import { Role } from '@merxy/types'

@Controller('uploads')
@UseGuards(JwtAuthGuard, RolesGuard)
export class UploadsController {
  constructor(private uploadsService: UploadsService) {}

  @Post('image')
  @Roles(Role.INVENTORY)
  @UseInterceptors(FileInterceptor('file'))
  async uploadImage(@UploadedFile() file: Express.Multer.File) {
    if (!file) throw new BadRequestException('No file provided')
    const key = await this.uploadsService.uploadProductImage(
      file.buffer,
      file.mimetype,
    )
    return { key, urls: {
      sm: this.uploadsService.getImageUrl(key, 'sm'),
      md: this.uploadsService.getImageUrl(key, 'md'),
      lg: this.uploadsService.getImageUrl(key, 'lg'),
    }}
  }
}
```

- [ ] **Step 4: Create apps/api/src/uploads/uploads.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { UploadsController } from './uploads.controller'
import { UploadsService } from './uploads.service'

@Module({
  controllers: [UploadsController],
  providers: [UploadsService],
  exports: [UploadsService],
})
export class UploadsModule {}
```

- [ ] **Step 5: Register in app.module.ts**

Add `UploadsModule` to imports in `app.module.ts`.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/uploads apps/api/src/app.module.ts
git commit -m "feat: add image upload service with Sharp resize and S3 storage"
```

---

## Task 3: Categories, Brands, Units modules

**Files:**
- Create: `apps/api/src/categories/` (module, controller, service, dto)
- Create: `apps/api/src/brands/` (module, controller, service)
- Create: `apps/api/src/units/` (module, controller, service)

- [ ] **Step 1: Create categories DTO**

Create `apps/api/src/categories/dto/create-category.dto.ts`:

```typescript
import { IsOptional, IsString, IsInt, Min } from 'class-validator'

export class CreateCategoryDto {
  @IsString()
  name: string

  @IsString()
  slug: string

  @IsOptional()
  @IsString()
  parentId?: string

  @IsOptional()
  @IsInt()
  @Min(0)
  sortOrder?: number
}
```

- [ ] **Step 2: Create apps/api/src/categories/categories.service.ts**

```typescript
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { CreateCategoryDto } from './dto/create-category.dto'

@Injectable()
export class CategoriesService {
  constructor(private prisma: PrismaService) {}

  findAll() {
    return this.prisma.category.findMany({
      include: { children: true },
      where: { parentId: null },
      orderBy: { sortOrder: 'asc' },
    })
  }

  async create(dto: CreateCategoryDto) {
    const existing = await this.prisma.category.findUnique({ where: { slug: dto.slug } })
    if (existing) throw new ConflictException('Slug already in use')
    return this.prisma.category.create({ data: dto })
  }

  async update(id: string, dto: Partial<CreateCategoryDto>) {
    await this.findById(id)
    return this.prisma.category.update({ where: { id }, data: dto })
  }

  async remove(id: string) {
    await this.findById(id)
    return this.prisma.category.delete({ where: { id } })
  }

  private async findById(id: string) {
    const cat = await this.prisma.category.findUnique({ where: { id } })
    if (!cat) throw new NotFoundException('Category not found')
    return cat
  }
}
```

- [ ] **Step 3: Create apps/api/src/categories/categories.controller.ts**

```typescript
import { Body, Controller, Delete, Get, Param, Patch, Post, UseGuards } from '@nestjs/common'
import { CategoriesService } from './categories.service'
import { CreateCategoryDto } from './dto/create-category.dto'
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard'
import { RolesGuard } from '../auth/guards/roles.guard'
import { Roles } from '../auth/decorators/roles.decorator'
import { Role } from '@merxy/types'

@Controller('categories')
@UseGuards(JwtAuthGuard, RolesGuard)
export class CategoriesController {
  constructor(private service: CategoriesService) {}

  @Get()
  findAll() { return this.service.findAll() }

  @Post()
  @Roles(Role.INVENTORY)
  create(@Body() dto: CreateCategoryDto) { return this.service.create(dto) }

  @Patch(':id')
  @Roles(Role.INVENTORY)
  update(@Param('id') id: string, @Body() dto: Partial<CreateCategoryDto>) {
    return this.service.update(id, dto)
  }

  @Delete(':id')
  @Roles(Role.ADMIN)
  remove(@Param('id') id: string) { return this.service.remove(id) }
}
```

- [ ] **Step 4: Create categories.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { CategoriesController } from './categories.controller'
import { CategoriesService } from './categories.service'

@Module({ controllers: [CategoriesController], providers: [CategoriesService], exports: [CategoriesService] })
export class CategoriesModule {}
```

- [ ] **Step 5: Create brands and units modules (same pattern)**

Create `apps/api/src/brands/brands.service.ts`:

```typescript
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class BrandsService {
  constructor(private prisma: PrismaService) {}
  findAll() { return this.prisma.brand.findMany({ orderBy: { name: 'asc' } }) }
  async create(name: string) {
    const exists = await this.prisma.brand.findUnique({ where: { name } })
    if (exists) throw new ConflictException('Brand name already exists')
    return this.prisma.brand.create({ data: { name } })
  }
  async update(id: string, name: string) {
    const exists = await this.prisma.brand.findUnique({ where: { id } })
    if (!exists) throw new NotFoundException('Brand not found')
    return this.prisma.brand.update({ where: { id }, data: { name } })
  }
  async remove(id: string) {
    const exists = await this.prisma.brand.findUnique({ where: { id } })
    if (!exists) throw new NotFoundException('Brand not found')
    return this.prisma.brand.delete({ where: { id } })
  }
}
```

Create `apps/api/src/brands/brands.controller.ts`:

```typescript
import { Body, Controller, Delete, Get, Param, Patch, Post, UseGuards } from '@nestjs/common'
import { BrandsService } from './brands.service'
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard'
import { RolesGuard } from '../auth/guards/roles.guard'
import { Roles } from '../auth/decorators/roles.decorator'
import { Role } from '@merxy/types'
import { IsString } from 'class-validator'

class BrandDto { @IsString() name: string }

@Controller('brands')
@UseGuards(JwtAuthGuard, RolesGuard)
export class BrandsController {
  constructor(private service: BrandsService) {}
  @Get() findAll() { return this.service.findAll() }
  @Post() @Roles(Role.INVENTORY) create(@Body() dto: BrandDto) { return this.service.create(dto.name) }
  @Patch(':id') @Roles(Role.INVENTORY) update(@Param('id') id: string, @Body() dto: BrandDto) { return this.service.update(id, dto.name) }
  @Delete(':id') @Roles(Role.ADMIN) remove(@Param('id') id: string) { return this.service.remove(id) }
}
```

Create `apps/api/src/brands/brands.module.ts`:
```typescript
import { Module } from '@nestjs/common'
import { BrandsController } from './brands.controller'
import { BrandsService } from './brands.service'
@Module({ controllers: [BrandsController], providers: [BrandsService], exports: [BrandsService] })
export class BrandsModule {}
```

Repeat the same pattern for Units — service/controller/module in `apps/api/src/units/`. Units have `name` and `abbreviation` fields.

`apps/api/src/units/units.service.ts`:
```typescript
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

class UnitInput { name: string; abbreviation: string }

@Injectable()
export class UnitsService {
  constructor(private prisma: PrismaService) {}
  findAll() { return this.prisma.unit.findMany({ orderBy: { name: 'asc' } }) }
  async create(dto: UnitInput) {
    const exists = await this.prisma.unit.findUnique({ where: { name: dto.name } })
    if (exists) throw new ConflictException('Unit name already exists')
    return this.prisma.unit.create({ data: dto })
  }
  async update(id: string, dto: Partial<UnitInput>) {
    const exists = await this.prisma.unit.findUnique({ where: { id } })
    if (!exists) throw new NotFoundException('Unit not found')
    return this.prisma.unit.update({ where: { id }, data: dto })
  }
  async remove(id: string) {
    const exists = await this.prisma.unit.findUnique({ where: { id } })
    if (!exists) throw new NotFoundException('Unit not found')
    return this.prisma.unit.delete({ where: { id } })
  }
}
```

- [ ] **Step 6: Register all new modules in app.module.ts**

```typescript
import { CategoriesModule } from './categories/categories.module'
import { BrandsModule } from './brands/brands.module'
import { UnitsModule } from './units/units.module'
// add to imports array
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/categories apps/api/src/brands apps/api/src/units apps/api/src/app.module.ts
git commit -m "feat: add categories, brands, and units CRUD modules"
```

---

## Task 4: Products service + controller (TDD)

**Files:**
- Create: `apps/api/src/products/dto/create-product.dto.ts`
- Create: `apps/api/src/products/products.service.ts`
- Create: `apps/api/src/products/products.service.spec.ts`
- Create: `apps/api/src/products/products.controller.ts`
- Create: `apps/api/src/products/products.module.ts`

- [ ] **Step 1: Create create-product.dto.ts**

```typescript
import { IsString, IsOptional, IsInt, Min, IsBoolean, IsArray } from 'class-validator'

export class CreateProductDto {
  @IsString()
  name: string

  @IsOptional()
  @IsString()
  sku?: string

  @IsOptional()
  @IsString()
  barcode?: string

  @IsOptional()
  @IsString()
  categoryId?: string

  @IsOptional()
  @IsString()
  brandId?: string

  @IsOptional()
  @IsString()
  unitId?: string

  @IsInt()
  @Min(0)
  costPrice: number

  @IsInt()
  @Min(0)
  sellPrice: number

  @IsOptional()
  @IsArray()
  imageUrls?: string[]

  @IsOptional()
  @IsInt()
  @Min(0)
  mainImageIndex?: number

  @IsOptional()
  @IsBoolean()
  isActive?: boolean

  @IsOptional()
  @IsBoolean()
  isTrackable?: boolean
}
```

- [ ] **Step 2: Write failing tests**

Create `apps/api/src/products/products.service.spec.ts`:

```typescript
import { Test, TestingModule } from '@nestjs/testing'
import { ConflictException, NotFoundException } from '@nestjs/common'
import { ProductsService } from './products.service'
import { PrismaService } from '../prisma/prisma.service'

const mockPrisma = {
  product: {
    findMany: jest.fn(),
    findUnique: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
  },
}

describe('ProductsService', () => {
  let service: ProductsService

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ProductsService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile()
    service = module.get<ProductsService>(ProductsService)
    jest.clearAllMocks()
  })

  describe('create', () => {
    it('throws ConflictException when SKU already exists', async () => {
      mockPrisma.product.findUnique.mockResolvedValue({ id: 'p1' })
      await expect(service.create({ name: 'Test', costPrice: 100, sellPrice: 200, sku: 'SKU-1' }))
        .rejects.toThrow(ConflictException)
    })

    it('creates product and returns it', async () => {
      mockPrisma.product.findUnique.mockResolvedValue(null)
      const product = { id: 'p1', name: 'Tea', sku: 'TEA-1', sellPrice: 1500 }
      mockPrisma.product.create.mockResolvedValue(product)
      const result = await service.create({ name: 'Tea', costPrice: 900, sellPrice: 1500 })
      expect(mockPrisma.product.create).toHaveBeenCalled()
      expect(result).toEqual(product)
    })
  })

  describe('update', () => {
    it('throws NotFoundException when product does not exist', async () => {
      mockPrisma.product.findUnique.mockResolvedValue(null)
      await expect(service.update('p999', { name: 'New' })).rejects.toThrow(NotFoundException)
    })

    it('updates and returns product', async () => {
      mockPrisma.product.findUnique.mockResolvedValue({ id: 'p1' })
      mockPrisma.product.update.mockResolvedValue({ id: 'p1', name: 'New' })
      const result = await service.update('p1', { name: 'New' })
      expect(result).toEqual({ id: 'p1', name: 'New' })
    })
  })
})
```

- [ ] **Step 3: Run tests — verify FAIL**

```bash
cd apps/api && pnpm test products.service
```

Expected: FAIL — `ProductsService` not found.

- [ ] **Step 4: Create apps/api/src/products/products.service.ts**

```typescript
import {
  ConflictException,
  Injectable,
  NotFoundException,
} from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { CreateProductDto } from './dto/create-product.dto'
import { randomUUID } from 'crypto'

const PRODUCT_INCLUDE = {
  category: { select: { id: true, name: true } },
  brand: { select: { id: true, name: true } },
  unit: { select: { id: true, name: true, abbreviation: true } },
}

@Injectable()
export class ProductsService {
  constructor(private prisma: PrismaService) {}

  findAll(params: {
    search?: string
    categoryId?: string
    isActive?: boolean
    page?: number
    limit?: number
  } = {}) {
    const { search, categoryId, isActive, page = 1, limit = 50 } = params
    return this.prisma.product.findMany({
      where: {
        ...(search ? { OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { sku: { contains: search, mode: 'insensitive' } },
          { barcode: { contains: search, mode: 'insensitive' } },
        ]} : {}),
        ...(categoryId ? { categoryId } : {}),
        ...(isActive !== undefined ? { isActive } : {}),
      },
      include: PRODUCT_INCLUDE,
      orderBy: { name: 'asc' },
      skip: (page - 1) * limit,
      take: limit,
    })
  }

  async findById(id: string) {
    const product = await this.prisma.product.findUnique({
      where: { id },
      include: PRODUCT_INCLUDE,
    })
    if (!product) throw new NotFoundException('Product not found')
    return product
  }

  async findByBarcode(barcode: string) {
    const product = await this.prisma.product.findUnique({
      where: { barcode },
      include: PRODUCT_INCLUDE,
    })
    if (!product) throw new NotFoundException('Product not found')
    return product
  }

  async create(dto: CreateProductDto) {
    const sku = dto.sku ?? `SKU-${randomUUID().slice(0, 8).toUpperCase()}`

    if (dto.sku) {
      const existing = await this.prisma.product.findUnique({ where: { sku } })
      if (existing) throw new ConflictException('SKU already in use')
    }

    if (dto.barcode) {
      const existing = await this.prisma.product.findUnique({ where: { barcode: dto.barcode } })
      if (existing) throw new ConflictException('Barcode already in use')
    }

    return this.prisma.product.create({
      data: {
        ...dto,
        sku,
        imageUrls: dto.imageUrls ?? [],
        mainImageIndex: dto.mainImageIndex ?? 0,
        isActive: dto.isActive ?? true,
        isTrackable: dto.isTrackable ?? true,
      },
      include: PRODUCT_INCLUDE,
    })
  }

  async update(id: string, dto: Partial<CreateProductDto>) {
    const product = await this.prisma.product.findUnique({ where: { id } })
    if (!product) throw new NotFoundException('Product not found')
    return this.prisma.product.update({ where: { id }, data: dto, include: PRODUCT_INCLUDE })
  }

  async deactivate(id: string) {
    const product = await this.prisma.product.findUnique({ where: { id } })
    if (!product) throw new NotFoundException('Product not found')
    return this.prisma.product.update({ where: { id }, data: { isActive: false } })
  }
}
```

- [ ] **Step 5: Run tests — verify PASS**

```bash
cd apps/api && pnpm test products.service
```

Expected: All tests pass.

- [ ] **Step 6: Create apps/api/src/products/products.controller.ts**

```typescript
import {
  Body, Controller, Delete, Get, Param, Patch, Post, Query, UseGuards,
} from '@nestjs/common'
import { ProductsService } from './products.service'
import { CreateProductDto } from './dto/create-product.dto'
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard'
import { RolesGuard } from '../auth/guards/roles.guard'
import { Roles } from '../auth/decorators/roles.decorator'
import { Role } from '@merxy/types'

@Controller('products')
@UseGuards(JwtAuthGuard, RolesGuard)
export class ProductsController {
  constructor(private service: ProductsService) {}

  @Get()
  findAll(
    @Query('search') search?: string,
    @Query('categoryId') categoryId?: string,
    @Query('page') page?: string,
    @Query('limit') limit?: string,
  ) {
    return this.service.findAll({
      search,
      categoryId,
      page: page ? Number(page) : 1,
      limit: limit ? Number(limit) : 50,
    })
  }

  @Get('barcode/:barcode')
  findByBarcode(@Param('barcode') barcode: string) {
    return this.service.findByBarcode(barcode)
  }

  @Get(':id')
  findById(@Param('id') id: string) { return this.service.findById(id) }

  @Post()
  @Roles(Role.INVENTORY)
  create(@Body() dto: CreateProductDto) { return this.service.create(dto) }

  @Patch(':id')
  @Roles(Role.INVENTORY)
  update(@Param('id') id: string, @Body() dto: Partial<CreateProductDto>) {
    return this.service.update(id, dto)
  }

  @Delete(':id')
  @Roles(Role.INVENTORY)
  deactivate(@Param('id') id: string) { return this.service.deactivate(id) }
}
```

- [ ] **Step 7: Create apps/api/src/products/products.module.ts and register**

```typescript
import { Module } from '@nestjs/common'
import { ProductsController } from './products.controller'
import { ProductsService } from './products.service'

@Module({
  controllers: [ProductsController],
  providers: [ProductsService],
  exports: [ProductsService],
})
export class ProductsModule {}
```

Add `ProductsModule` to `app.module.ts` imports.

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/products apps/api/src/app.module.ts
git commit -m "feat: add Products module with CRUD and barcode lookup (TDD)"
```

---

## Task 5: Inventory service (StoreProduct + StockMovement) (TDD)

**Files:**
- Create: `apps/api/src/inventory/inventory.service.ts`
- Create: `apps/api/src/inventory/inventory.service.spec.ts`
- Create: `apps/api/src/inventory/inventory.controller.ts`
- Create: `apps/api/src/inventory/inventory.module.ts`

- [ ] **Step 1: Write failing inventory tests**

Create `apps/api/src/inventory/inventory.service.spec.ts`:

```typescript
import { Test } from '@nestjs/testing'
import { BadRequestException, NotFoundException } from '@nestjs/common'
import { InventoryService } from './inventory.service'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'
import { StockMovementType } from '@prisma/client'

const mockPrisma = {
  storeProduct: { findUnique: jest.fn(), upsert: jest.fn(), findMany: jest.fn(), update: jest.fn() },
  stockMovement: { create: jest.fn(), findMany: jest.fn() },
  $transaction: jest.fn(),
}
const mockRedis = { set: jest.fn() }

describe('InventoryService', () => {
  let service: InventoryService

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        InventoryService,
        { provide: PrismaService, useValue: mockPrisma },
        { provide: RedisService, useValue: mockRedis },
      ],
    }).compile()
    service = module.get<InventoryService>(InventoryService)
    jest.clearAllMocks()
  })

  describe('adjustStock', () => {
    it('throws NotFoundException when StoreProduct does not exist', async () => {
      mockPrisma.$transaction.mockImplementation(async (cb: any) => cb(mockPrisma))
      mockPrisma.storeProduct.findUnique.mockResolvedValue(null)
      await expect(service.adjustStock('store-1', 'product-1', 50, 'Correction', 'user-1'))
        .rejects.toThrow(NotFoundException)
    })

    it('writes StockMovement and updates StoreProduct atomically', async () => {
      const sp = { id: 'sp1', storeId: 'store-1', productId: 'p1', stockQty: 30 }
      mockPrisma.$transaction.mockImplementation(async (cb: any) => cb(mockPrisma))
      mockPrisma.storeProduct.findUnique.mockResolvedValue(sp)
      mockPrisma.stockMovement.create.mockResolvedValue({ id: 'mv1' })
      mockPrisma.storeProduct.update.mockResolvedValue({ ...sp, stockQty: 50 })

      await service.adjustStock('store-1', 'p1', 50, 'Correction', 'user-1')

      expect(mockPrisma.stockMovement.create).toHaveBeenCalledWith(
        expect.objectContaining({
          data: expect.objectContaining({ type: StockMovementType.ADJUST, qtyChange: 20 }),
        })
      )
      expect(mockPrisma.storeProduct.update).toHaveBeenCalledWith(
        expect.objectContaining({ data: { stockQty: 50 } })
      )
    })
  })
})
```

- [ ] **Step 2: Run tests — verify FAIL**

```bash
cd apps/api && pnpm test inventory.service
```

Expected: FAIL.

- [ ] **Step 3: Create apps/api/src/inventory/inventory.service.ts**

```typescript
import { Injectable, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'
import { StockMovementType } from '@prisma/client'

const LOW_STOCK_CHANNEL = 'stock:low'

@Injectable()
export class InventoryService {
  constructor(
    private prisma: PrismaService,
    private redis: RedisService,
  ) {}

  getStoreStock(storeId: string, search?: string) {
    return this.prisma.storeProduct.findMany({
      where: {
        storeId,
        product: search
          ? { name: { contains: search, mode: 'insensitive' } }
          : undefined,
      },
      include: {
        product: {
          include: { category: { select: { id: true, name: true } }, unit: true },
        },
      },
      orderBy: { product: { name: 'asc' } },
    })
  }

  getMovements(storeId: string, productId?: string, limit = 100) {
    return this.prisma.stockMovement.findMany({
      where: { storeId, ...(productId ? { productId } : {}) },
      include: {
        product: { select: { name: true, sku: true } },
        createdBy: { select: { name: true } },
      },
      orderBy: { createdAt: 'desc' },
      take: limit,
    })
  }

  /**
   * Set absolute qty for a product in a store.
   * Calculates the diff, writes a ADJUST movement, and updates stock_qty
   * in a single Prisma transaction.
   */
  async adjustStock(
    storeId: string,
    productId: string,
    newQty: number,
    reason: string,
    createdById: string,
  ) {
    return this.prisma.$transaction(async (tx) => {
      const sp = await tx.storeProduct.findUnique({
        where: { storeId_productId: { storeId, productId } },
      })
      if (!sp) throw new NotFoundException('Product not found in this store')

      const qtyChange = newQty - sp.stockQty

      await tx.stockMovement.create({
        data: {
          storeId,
          productId,
          type: StockMovementType.ADJUST,
          qtyChange,
          note: reason,
          createdById,
        },
      })

      const updated = await tx.storeProduct.update({
        where: { storeId_productId: { storeId, productId } },
        data: { stockQty: newQty },
      })

      if (updated.stockQty <= updated.minStockQty) {
        await this.redis.set(
          `${LOW_STOCK_CHANNEL}:${storeId}:${productId}`,
          JSON.stringify({ storeId, productId, qty: updated.stockQty }),
          60,
        )
      }

      return updated
    })
  }

  /**
   * Initialize stock for a newly added product in a store.
   */
  async initializeStock(
    storeId: string,
    productId: string,
    initialQty: number,
    createdById: string,
  ) {
    return this.prisma.$transaction(async (tx) => {
      const sp = await tx.storeProduct.upsert({
        where: { storeId_productId: { storeId, productId } },
        create: { storeId, productId, stockQty: 0, minStockQty: 5 },
        update: {},
      })

      if (initialQty > 0) {
        await tx.stockMovement.create({
          data: {
            storeId,
            productId,
            type: StockMovementType.RECEIVE,
            qtyChange: initialQty,
            note: 'Initial stock',
            createdById,
          },
        })
        await tx.storeProduct.update({
          where: { storeId_productId: { storeId, productId } },
          data: { stockQty: initialQty },
        })
      }

      return sp
    })
  }

  /**
   * Deduct stock on sale — called within an order transaction.
   */
  async deductForSale(
    tx: any,
    storeId: string,
    productId: string,
    qty: number,
    orderId: string,
    createdById: string,
  ) {
    const sp = await tx.storeProduct.findUnique({
      where: { storeId_productId: { storeId, productId } },
    })
    if (!sp) throw new NotFoundException(`Product ${productId} not in store`)

    const newQty = sp.stockQty - qty
    await tx.stockMovement.create({
      data: {
        storeId, productId,
        type: StockMovementType.SALE,
        qtyChange: -qty,
        referenceId: orderId,
        createdById,
      },
    })
    return tx.storeProduct.update({
      where: { storeId_productId: { storeId, productId } },
      data: { stockQty: newQty },
    })
  }
}
```

- [ ] **Step 4: Run tests — verify PASS**

```bash
cd apps/api && pnpm test inventory.service
```

Expected: All tests pass.

- [ ] **Step 5: Create apps/api/src/inventory/inventory.controller.ts**

```typescript
import { Body, Controller, Get, Param, Post, Query, UseGuards } from '@nestjs/common'
import { InventoryService } from './inventory.service'
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard'
import { RolesGuard } from '../auth/guards/roles.guard'
import { Roles } from '../auth/decorators/roles.decorator'
import { CurrentUser } from '../auth/decorators/current-user.decorator'
import { Role, AuthUser } from '@merxy/types'
import { IsInt, IsString, Min } from 'class-validator'

class AdjustDto {
  @IsInt() @Min(0) newQty: number
  @IsString() reason: string
}

class InitStockDto {
  @IsString() productId: string
  @IsInt() @Min(0) initialQty: number
}

@Controller('stores/:storeId/inventory')
@UseGuards(JwtAuthGuard, RolesGuard)
export class InventoryController {
  constructor(private service: InventoryService) {}

  @Get()
  @Roles(Role.INVENTORY)
  getStock(@Param('storeId') storeId: string, @Query('search') search?: string) {
    return this.service.getStoreStock(storeId, search)
  }

  @Get('movements')
  @Roles(Role.INVENTORY)
  getMovements(
    @Param('storeId') storeId: string,
    @Query('productId') productId?: string,
  ) {
    return this.service.getMovements(storeId, productId)
  }

  @Post('adjust/:productId')
  @Roles(Role.INVENTORY)
  adjust(
    @Param('storeId') storeId: string,
    @Param('productId') productId: string,
    @Body() dto: AdjustDto,
    @CurrentUser() user: AuthUser,
  ) {
    return this.service.adjustStock(storeId, productId, dto.newQty, dto.reason, user.id)
  }

  @Post('init')
  @Roles(Role.INVENTORY)
  init(
    @Param('storeId') storeId: string,
    @Body() dto: InitStockDto,
    @CurrentUser() user: AuthUser,
  ) {
    return this.service.initializeStock(storeId, dto.productId, dto.initialQty, user.id)
  }
}
```

- [ ] **Step 6: Create inventory.module.ts and register**

```typescript
import { Module } from '@nestjs/common'
import { InventoryController } from './inventory.controller'
import { InventoryService } from './inventory.service'

@Module({
  controllers: [InventoryController],
  providers: [InventoryService],
  exports: [InventoryService],
})
export class InventoryModule {}
```

Add `InventoryModule` to `app.module.ts`.

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/inventory apps/api/src/app.module.ts
git commit -m "feat: add InventoryService with atomic stock adjustment and ledger (TDD)"
```

---

## Task 6: Stock panel — Products list page

**Files:**
- Create: `apps/web/src/app/(stock)/stock/products/page.tsx`
- Create: `apps/web/src/app/(stock)/stock/products/_components/ProductsTable.tsx`
- Create: `apps/web/src/app/(stock)/stock/products/_components/ProductDrawer.tsx`
- Create: `apps/web/src/app/(stock)/stock/products/_components/ImageUploader.tsx`
- Create: `apps/web/src/hooks/useProducts.ts`

- [ ] **Step 1: Install frontend dependencies**

```bash
cd apps/web && pnpm add @tanstack/react-table react-hook-form @hookform/resolvers zod
```

- [ ] **Step 2: Create apps/web/src/hooks/useProducts.ts**

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { api } from '@/lib/api'
import { Product, CreateProductDto } from '@merxy/types'

export function useProducts(params?: { search?: string; categoryId?: string }) {
  return useQuery({
    queryKey: ['products', params],
    queryFn: async () => {
      const { data } = await api.get<{ data: Product[] }>('/products', { params })
      return data.data ?? []
    },
  })
}

export function useCreateProduct() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: async (dto: CreateProductDto) => {
      const { data } = await api.post<{ data: Product }>('/products', dto)
      return data.data!
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ['products'] }),
  })
}

export function useUpdateProduct() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: async ({ id, dto }: { id: string; dto: Partial<CreateProductDto> }) => {
      const { data } = await api.patch<{ data: Product }>(`/products/${id}`, dto)
      return data.data!
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ['products'] }),
  })
}
```

- [ ] **Step 3: Create ImageUploader.tsx**

```tsx
'use client'

import { useState, useRef } from 'react'
import { api } from '@/lib/api'

interface ImageUploaderProps {
  value: string[]
  mainIndex: number
  onChange: (urls: string[], mainIndex: number) => void
}

export function ImageUploader({ value, mainIndex, onChange }: ImageUploaderProps) {
  const [uploading, setUploading] = useState(false)
  const inputRef = useRef<HTMLInputElement>(null)

  async function handleFiles(files: FileList | null) {
    if (!files || value.length >= 5) return
    setUploading(true)
    try {
      const toUpload = Array.from(files).slice(0, 5 - value.length)
      const newUrls: string[] = []
      for (const file of toUpload) {
        const form = new FormData()
        form.append('file', file)
        const { data } = await api.post<{ data: { urls: { md: string } } }>('/uploads/image', form, {
          headers: { 'Content-Type': 'multipart/form-data' },
        })
        newUrls.push(data.data!.urls.md)
      }
      onChange([...value, ...newUrls], mainIndex)
    } finally {
      setUploading(false)
    }
  }

  function removeImage(index: number) {
    const updated = value.filter((_, i) => i !== index)
    onChange(updated, Math.min(mainIndex, updated.length - 1))
  }

  function setMain(index: number) {
    onChange(value, index)
  }

  return (
    <div className="space-y-3">
      <div
        onClick={() => inputRef.current?.click()}
        onDragOver={(e) => e.preventDefault()}
        onDrop={(e) => { e.preventDefault(); handleFiles(e.dataTransfer.files) }}
        className="border-2 border-dashed border-border rounded-xl p-6 text-center cursor-pointer hover:border-primary hover:bg-primary-tint transition-colors"
      >
        {uploading ? (
          <p className="text-text-muted text-sm">Uploading…</p>
        ) : (
          <>
            <p className="text-text-muted text-sm font-semibold">Click to upload or drag & drop</p>
            <p className="text-[11px] text-text-muted mt-1">JPG, PNG, WEBP · Max 2MB · Up to 5 images</p>
          </>
        )}
      </div>
      <input
        ref={inputRef}
        type="file"
        accept="image/jpeg,image/png,image/webp"
        multiple
        className="hidden"
        onChange={(e) => handleFiles(e.target.files)}
      />
      {value.length > 0 && (
        <div className="flex gap-2 flex-wrap">
          {value.map((url, i) => (
            <div
              key={url}
              className={`relative w-16 h-16 rounded-lg border-2 overflow-hidden cursor-pointer ${
                i === mainIndex ? 'border-primary' : 'border-border'
              }`}
              onClick={() => setMain(i)}
            >
              {/* eslint-disable-next-line @next/next/no-img-element */}
              <img src={url} alt="" className="w-full h-full object-cover" />
              {i === mainIndex && (
                <div className="absolute bottom-0 left-0 right-0 bg-primary text-white text-[8px] font-bold text-center py-0.5">
                  Main
                </div>
              )}
              <button
                type="button"
                onClick={(e) => { e.stopPropagation(); removeImage(i) }}
                className="absolute top-0.5 right-0.5 w-4 h-4 bg-black/50 rounded-full text-white text-[9px] flex items-center justify-center"
              >
                ✕
              </button>
            </div>
          ))}
          {value.length < 5 && (
            <div
              onClick={() => inputRef.current?.click()}
              className="w-16 h-16 rounded-lg border-2 border-dashed border-border flex items-center justify-center text-text-muted text-2xl cursor-pointer hover:border-primary"
            >
              +
            </div>
          )}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 4: Create ProductDrawer.tsx**

```tsx
'use client'

import { useEffect } from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Product } from '@merxy/types'
import { ImageUploader } from './ImageUploader'
import { useCreateProduct, useUpdateProduct } from '@/hooks/useProducts'

const schema = z.object({
  name: z.string().min(1, 'Name is required'),
  sku: z.string().optional(),
  barcode: z.string().optional(),
  categoryId: z.string().optional(),
  brandId: z.string().optional(),
  unitId: z.string().optional(),
  costPrice: z.coerce.number().min(0),
  sellPrice: z.coerce.number().min(1, 'Sell price required'),
  imageUrls: z.array(z.string()).default([]),
  mainImageIndex: z.number().default(0),
  isTrackable: z.boolean().default(true),
  isActive: z.boolean().default(true),
})

type FormValues = z.infer<typeof schema>

interface ProductDrawerProps {
  product?: Product | null
  open: boolean
  onClose: () => void
}

export function ProductDrawer({ product, open, onClose }: ProductDrawerProps) {
  const create = useCreateProduct()
  const update = useUpdateProduct()
  const isEdit = !!product

  const { register, handleSubmit, setValue, watch, reset, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { imageUrls: [], mainImageIndex: 0, isTrackable: true, isActive: true, costPrice: 0, sellPrice: 0 },
  })

  useEffect(() => {
    if (product) {
      reset({
        name: product.name, sku: product.sku ?? '', barcode: product.barcode ?? '',
        categoryId: product.categoryId ?? '', brandId: product.brandId ?? '',
        unitId: product.unitId ?? '', costPrice: product.costPrice, sellPrice: product.sellPrice,
        imageUrls: product.imageUrls, mainImageIndex: product.mainImageIndex,
        isTrackable: product.isTrackable, isActive: product.isActive,
      })
    } else {
      reset()
    }
  }, [product, reset])

  async function onSubmit(values: FormValues) {
    if (isEdit) {
      await update.mutateAsync({ id: product!.id, dto: values })
    } else {
      await create.mutateAsync(values)
    }
    onClose()
  }

  const imageUrls = watch('imageUrls')
  const mainImageIndex = watch('mainImageIndex')
  const isPending = create.isPending || update.isPending

  if (!open) return null

  return (
    <div className="fixed inset-0 z-50 flex justify-end">
      <div className="fixed inset-0 bg-black/30" onClick={onClose} />
      <div className="relative w-full max-w-[480px] bg-surface h-full flex flex-col shadow-2xl overflow-hidden">
        <div className="flex items-center justify-between px-5 py-4 border-b border-border bg-white">
          <h2 className="font-bold text-text-dark">{isEdit ? 'Edit Product' : 'Add Product'}</h2>
          <button onClick={onClose} className="w-8 h-8 bg-canvas rounded-lg flex items-center justify-center text-text-muted hover:text-text-dark min-h-touch min-w-touch">✕</button>
        </div>

        <form onSubmit={handleSubmit(onSubmit)} className="flex-1 overflow-y-auto px-5 py-4 space-y-4">
          <div>
            <label className="block text-[11px] font-bold text-[#4a5568] uppercase tracking-wide mb-1.5">Product Image</label>
            <ImageUploader
              value={imageUrls}
              mainIndex={mainImageIndex}
              onChange={(urls, idx) => { setValue('imageUrls', urls); setValue('mainImageIndex', idx) }}
            />
          </div>

          <div>
            <label className="block text-[11px] font-bold text-[#4a5568] uppercase tracking-wide mb-1.5">Name <span className="text-danger">*</span></label>
            <input {...register('name')} className="w-full px-3 py-2.5 border border-border rounded-lg text-sm focus:outline-none focus:border-primary min-h-touch" />
            {errors.name && <p className="text-danger text-xs mt-1">{errors.name.message}</p>}
          </div>

          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="block text-[11px] font-bold text-[#4a5568] uppercase tracking-wide mb-1.5">SKU</label>
              <input {...register('sku')} placeholder="Auto-generated" className="w-full px-3 py-2.5 border border-border rounded-lg text-sm focus:outline-none focus:border-primary min-h-touch" />
            </div>
            <div>
              <label className="block text-[11px] font-bold text-[#4a5568] uppercase tracking-wide mb-1.5">Barcode</label>
              <input {...register('barcode')} className="w-full px-3 py-2.5 border border-border rounded-lg text-sm focus:outline-none focus:border-primary min-h-touch" />
            </div>
          </div>

          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="block text-[11px] font-bold text-[#4a5568] uppercase tracking-wide mb-1.5">Cost Price</label>
              <div className="flex">
                <input type="number" {...register('costPrice')} className="flex-1 px-3 py-2.5 border border-border rounded-l-lg text-sm focus:outline-none focus:border-primary min-h-touch" />
                <span className="px-3 py-2.5 bg-canvas border border-l-0 border-border rounded-r-lg text-xs font-bold text-text-muted">Ks</span>
              </div>
            </div>
            <div>
              <label className="block text-[11px] font-bold text-[#4a5568] uppercase tracking-wide mb-1.5">Sell Price <span className="text-danger">*</span></label>
              <div className="flex">
                <input type="number" {...register('sellPrice')} className="flex-1 px-3 py-2.5 border border-border rounded-l-lg text-sm focus:outline-none focus:border-primary min-h-touch" />
                <span className="px-3 py-2.5 bg-canvas border border-l-0 border-border rounded-r-lg text-xs font-bold text-text-muted">Ks</span>
              </div>
              {errors.sellPrice && <p className="text-danger text-xs mt-1">{errors.sellPrice.message}</p>}
            </div>
          </div>

          <div className="flex items-center justify-between p-3 bg-canvas border border-border rounded-lg">
            <div>
              <div className="text-sm font-semibold text-text-dark">Track Inventory</div>
              <div className="text-xs text-text-muted">Deduct stock on each sale</div>
            </div>
            <button
              type="button"
              onClick={() => setValue('isTrackable', !watch('isTrackable'))}
              className={`w-9 h-5 rounded-full transition-colors relative ${watch('isTrackable') ? 'bg-primary' : 'bg-border'}`}
            >
              <span className={`absolute top-0.5 w-4 h-4 bg-white rounded-full transition-all ${watch('isTrackable') ? 'right-0.5' : 'left-0.5'}`} />
            </button>
          </div>
        </form>

        <div className="px-5 py-4 border-t border-border bg-white flex gap-3 justify-end">
          <button type="button" onClick={onClose} className="btn-ghost">Cancel</button>
          <button
            type="submit"
            form="product-form"
            onClick={handleSubmit(onSubmit)}
            disabled={isPending}
            className="btn-primary disabled:opacity-60"
          >
            {isPending ? 'Saving…' : 'Save Product'}
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Create stock/products/page.tsx**

```tsx
'use client'

import { useState } from 'react'
import { useProducts } from '@/hooks/useProducts'
import { ProductDrawer } from './_components/ProductDrawer'
import { TopBar } from '@/components/layout/TopBar'
import { formatKs } from '@merxy/utils'
import { Product } from '@merxy/types'

export default function ProductsPage() {
  const [search, setSearch] = useState('')
  const [drawerOpen, setDrawerOpen] = useState(false)
  const [editing, setEditing] = useState<Product | null>(null)
  const { data: products = [], isLoading } = useProducts({ search })

  function openCreate() { setEditing(null); setDrawerOpen(true) }
  function openEdit(p: Product) { setEditing(p); setDrawerOpen(true) }

  return (
    <>
      <TopBar
        title="Products"
        actions={
          <>
            <button className="btn-ghost text-xs">Import CSV</button>
            <button onClick={openCreate} className="btn-primary text-xs">+ Add Product</button>
          </>
        }
      />
      <div className="flex-1 overflow-auto p-5">
        <div className="flex gap-3 mb-4">
          <input
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="Search products…"
            className="px-3 py-2 border border-border rounded-lg text-sm bg-surface focus:outline-none focus:border-primary w-56 min-h-touch"
          />
        </div>

        <div className="bg-surface border border-border rounded-xl overflow-hidden">
          {isLoading ? (
            <div className="p-8 text-center text-text-muted text-sm">Loading…</div>
          ) : products.length === 0 ? (
            <div className="p-8 text-center text-text-muted text-sm">No products found</div>
          ) : (
            <table className="data-table w-full">
              <thead>
                <tr>
                  <th>Product</th><th>SKU</th><th>Category</th>
                  <th>Sell Price</th><th>Cost Price</th><th>Status</th><th></th>
                </tr>
              </thead>
              <tbody>
                {products.map((p) => (
                  <tr key={p.id} className="hover:bg-canvas transition-colors">
                    <td className="font-semibold">{p.name}</td>
                    <td className="font-mono text-xs text-text-muted">{p.sku}</td>
                    <td className="text-xs">{p.category?.name ?? '—'}</td>
                    <td className="font-semibold">{formatKs(p.sellPrice)}</td>
                    <td className="text-text-muted">{formatKs(p.costPrice)}</td>
                    <td>
                      <span className={`badge ${p.isActive ? 'badge-green' : 'badge-gray'}`}>
                        {p.isActive ? 'Active' : 'Inactive'}
                      </span>
                    </td>
                    <td>
                      <button onClick={() => openEdit(p)} className="btn-ghost text-xs px-3 py-1.5 min-h-touch">Edit</button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          )}
        </div>
      </div>

      <ProductDrawer
        product={editing}
        open={drawerOpen}
        onClose={() => setDrawerOpen(false)}
      />
    </>
  )
}
```

- [ ] **Step 6: Add TanStack Query provider to root layout**

Update `apps/web/src/app/layout.tsx`:

```tsx
'use client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useState } from 'react'
import type { Metadata } from 'next'
import './globals.css'

// Note: metadata export removed since this is now a client component.
// Move metadata to a separate server layout wrapper if needed.

export default function RootLayout({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient())
  return (
    <html lang="en">
      <body>
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      </body>
    </html>
  )
}
```

- [ ] **Step 7: Commit**

```bash
git add apps/web/src/app/\(stock\) apps/web/src/hooks apps/web/src/app/layout.tsx
git commit -m "feat: add Stock panel products list and add/edit drawer with image upload"
```

---

## Task 7: Stock overview + movement history pages

**Files:**
- Create: `apps/web/src/app/(stock)/stock/overview/page.tsx`
- Create: `apps/web/src/app/(stock)/stock/movements/page.tsx`
- Create: `apps/web/src/hooks/useInventory.ts`

- [ ] **Step 1: Create apps/web/src/hooks/useInventory.ts**

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { api } from '@/lib/api'
import { useAuthStore } from '@/store/auth.store'
import { StoreProduct, StockMovement } from '@merxy/types'

export function useStoreStock(search?: string) {
  const storeId = useAuthStore((s) => s.storeId)
  return useQuery({
    queryKey: ['stock', storeId, search],
    queryFn: async () => {
      const { data } = await api.get<{ data: StoreProduct[] }>(
        `/stores/${storeId}/inventory`,
        { params: { search } },
      )
      return data.data ?? []
    },
    enabled: !!storeId,
  })
}

export function useStockMovements(productId?: string) {
  const storeId = useAuthStore((s) => s.storeId)
  return useQuery({
    queryKey: ['movements', storeId, productId],
    queryFn: async () => {
      const { data } = await api.get<{ data: StockMovement[] }>(
        `/stores/${storeId}/inventory/movements`,
        { params: { productId } },
      )
      return data.data ?? []
    },
    enabled: !!storeId,
  })
}

export function useAdjustStock() {
  const storeId = useAuthStore((s) => s.storeId)
  const qc = useQueryClient()
  return useMutation({
    mutationFn: async ({ productId, newQty, reason }: { productId: string; newQty: number; reason: string }) => {
      const { data } = await api.post(`/stores/${storeId}/inventory/adjust/${productId}`, { newQty, reason })
      return data.data
    },
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['stock', storeId] })
      qc.invalidateQueries({ queryKey: ['movements', storeId] })
    },
  })
}
```

- [ ] **Step 2: Create stock/overview/page.tsx**

```tsx
'use client'

import { useState } from 'react'
import { useStoreStock } from '@/hooks/useInventory'
import { TopBar } from '@/components/layout/TopBar'
import { formatKs } from '@merxy/utils'

export default function StockOverviewPage() {
  const [search, setSearch] = useState('')
  const { data: stock = [], isLoading } = useStoreStock(search)
  const lowStock = stock.filter((s) => s.stockQty <= s.minStockQty)

  return (
    <>
      <TopBar title="Stock Overview" />
      <div className="flex-1 overflow-auto p-5">
        {lowStock.length > 0 && (
          <div className="mb-4 p-3 bg-[#fef3c7] border border-[#fde68a] rounded-xl text-sm text-[#92400e] font-medium">
            ⚠ {lowStock.length} product{lowStock.length > 1 ? 's' : ''} at or below minimum stock level
          </div>
        )}
        <div className="flex gap-3 mb-4">
          <input value={search} onChange={(e) => setSearch(e.target.value)}
            placeholder="Search products…"
            className="px-3 py-2 border border-border rounded-lg text-sm bg-surface focus:outline-none focus:border-primary w-56 min-h-touch" />
        </div>
        <div className="bg-surface border border-border rounded-xl overflow-hidden">
          {isLoading ? <div className="p-8 text-center text-text-muted text-sm">Loading…</div> : (
            <table className="data-table w-full">
              <thead>
                <tr><th>Product</th><th>Category</th><th>Stock Qty</th><th>Min</th><th>Sell Price</th><th>Status</th></tr>
              </thead>
              <tbody>
                {stock.map((s) => {
                  const isLow = s.stockQty <= s.minStockQty
                  const isOut = s.stockQty === 0
                  return (
                    <tr key={s.id} className="hover:bg-canvas">
                      <td className="font-semibold">{s.product?.name}</td>
                      <td className="text-xs text-text-muted">{s.product?.category?.name ?? '—'}</td>
                      <td className={`font-bold ${isOut ? 'text-danger' : isLow ? 'text-warning' : 'text-text-dark'}`}>
                        {s.stockQty} {isLow && !isOut ? '⚠' : ''}</td>
                      <td className="text-text-muted">{s.minStockQty}</td>
                      <td>{s.product ? formatKs(s.product.sellPrice) : '—'}</td>
                      <td>
                        <span className={`badge ${isOut ? 'badge-red' : isLow ? 'badge-yellow' : 'badge-green'}`}>
                          {isOut ? 'Out of Stock' : isLow ? 'Low Stock' : 'In Stock'}
                        </span>
                      </td>
                    </tr>
                  )
                })}
              </tbody>
            </table>
          )}
        </div>
      </div>
    </>
  )
}
```

- [ ] **Step 3: Create stock/movements/page.tsx**

```tsx
'use client'

import { useStockMovements } from '@/hooks/useInventory'
import { TopBar } from '@/components/layout/TopBar'
import { formatDateTime } from '@merxy/utils'

const TYPE_LABELS: Record<string, string> = {
  SALE: 'Sale', RECEIVE: 'Receive', ADJUST: 'Adjustment',
  TRANSFER_OUT: 'Transfer Out', TRANSFER_IN: 'Transfer In',
}

export default function MovementsPage() {
  const { data: movements = [], isLoading } = useStockMovements()

  return (
    <>
      <TopBar title="Stock Movements" />
      <div className="flex-1 overflow-auto p-5">
        <div className="bg-surface border border-border rounded-xl overflow-hidden">
          {isLoading ? <div className="p-8 text-center text-text-muted text-sm">Loading…</div> : (
            <table className="data-table w-full">
              <thead>
                <tr><th>Date</th><th>Product</th><th>Type</th><th>Qty Change</th><th>Note</th><th>By</th></tr>
              </thead>
              <tbody>
                {movements.map((m) => (
                  <tr key={m.id} className="hover:bg-canvas">
                    <td className="text-xs text-text-muted">{formatDateTime(m.createdAt)}</td>
                    <td className="font-semibold text-sm">{m.product?.name}</td>
                    <td><span className="badge badge-blue">{TYPE_LABELS[m.type] ?? m.type}</span></td>
                    <td className={`font-bold ${m.qtyChange < 0 ? 'text-danger' : 'text-primary'}`}>
                      {m.qtyChange > 0 ? '+' : ''}{m.qtyChange}
                    </td>
                    <td className="text-xs text-text-muted">{m.note ?? '—'}</td>
                    <td className="text-xs">{m.createdBy?.name}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          )}
        </div>
      </div>
    </>
  )
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/app/\(stock\) apps/web/src/hooks/useInventory.ts
git commit -m "feat: add stock overview and movement history pages"
```

---

## Task 8: Run all tests + push

- [ ] **Step 1: Run all API tests**

```bash
cd apps/api && pnpm test
```

Expected: All tests pass.

- [ ] **Step 2: Run API coverage check**

```bash
cd apps/api && pnpm test:cov
```

Expected: Coverage ≥ 80% on products and inventory modules.

- [ ] **Step 3: Push**

```bash
cd ../.. && git push
```

---

## Phase 2 Complete

At this point you have:
- Global product catalog with CRUD, barcode lookup, image upload (S3 + Sharp)
- Per-store stock levels via `StoreProduct`
- Append-only `StockMovement` ledger with atomic transactions
- Categories, Brands, Units management
- Stock panel frontend: product list, add/edit drawer with image upload, stock overview, movement history

**Next: Phase 3 — POS Selling** (`docs/superpowers/plans/2026-04-18-phase-3-pos-selling.md`)
