# Phase 7 — Advanced Inventory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build advanced inventory features — suppliers, purchase orders (receive stock), inter-store stock transfers, stock count (physical count reconciliation), and low-stock alerts via BullMQ.

**Architecture:** Four new NestJS modules: `SuppliersModule`, `PurchaseOrdersModule`, `TransfersModule`, and `StockCountModule`. A BullMQ worker (`LowStockWorker`) runs after every stock-deducting transaction to check `stockQty < minStockQty` and emits a `stock:low` WebSocket event + stores a notification. The frontend adds stock panel pages for each feature.

**Tech Stack:** NestJS modules, Prisma transactions, BullMQ (Redis-backed), Socket.IO, Next.js stock panel pages, React Hook Form + Zod

**Depends on:** Phase 2 complete (inventory, products), Phase 3 complete (WebSocket gateway, orders)

---

## File Map

```
apps/api/src/
├── prisma/schema.prisma                       # add Supplier, PurchaseOrder, PurchaseOrderItem,
│                                              #     StockTransfer, StockTransferItem, StockCount,
│                                              #     StockCountItem
├── suppliers/
│   ├── suppliers.module.ts
│   ├── suppliers.controller.ts
│   └── suppliers.service.ts
├── purchase-orders/
│   ├── purchase-orders.module.ts
│   ├── purchase-orders.controller.ts
│   ├── purchase-orders.service.ts
│   └── purchase-orders.service.spec.ts
├── transfers/
│   ├── transfers.module.ts
│   ├── transfers.controller.ts
│   ├── transfers.service.ts
│   └── transfers.service.spec.ts
├── stock-count/
│   ├── stock-count.module.ts
│   ├── stock-count.controller.ts
│   └── stock-count.service.ts
└── low-stock/
    ├── low-stock.module.ts
    ├── low-stock.processor.ts                 # BullMQ worker
    └── low-stock.service.ts                   # enqueues jobs

apps/web/src/app/(stock)/stock/
├── purchase-orders/page.tsx
├── transfers/page.tsx
├── count/page.tsx
└── low-stock/page.tsx
```

---

## Task 1: Extend Prisma schema — Supplier, PurchaseOrder, StockTransfer, StockCount

- [ ] **Step 1: Append models to schema.prisma**

```prisma
model Supplier {
  id           String          @id @default(cuid())
  name         String
  contactName  String?         @map("contact_name")
  email        String?
  phone        String?
  address      String?
  createdAt    DateTime        @default(now()) @map("created_at")

  purchaseOrders PurchaseOrder[]

  @@map("suppliers")
}

enum PurchaseOrderStatus {
  DRAFT
  ORDERED
  PARTIAL
  RECEIVED
  CANCELLED
}

model PurchaseOrder {
  id          String              @id @default(cuid())
  storeId     String              @map("store_id")
  supplierId  String              @map("supplier_id")
  status      PurchaseOrderStatus @default(DRAFT)
  totalAmount Int                 @default(0) @map("total_amount")
  notes       String?
  createdById String              @map("created_by_id")
  createdAt   DateTime            @default(now()) @map("created_at")

  store       Store               @relation(fields: [storeId], references: [id])
  supplier    Supplier            @relation(fields: [supplierId], references: [id])
  createdBy   User                @relation(fields: [createdById], references: [id])
  items       PurchaseOrderItem[]

  @@map("purchase_orders")
}

model PurchaseOrderItem {
  id              String        @id @default(cuid())
  purchaseOrderId String        @map("purchase_order_id")
  productId       String        @map("product_id")
  qty             Int
  costPrice       Int           @map("cost_price")
  receivedQty     Int           @default(0) @map("received_qty")

  purchaseOrder   PurchaseOrder @relation(fields: [purchaseOrderId], references: [id])
  product         Product       @relation(fields: [productId], references: [id])

  @@map("purchase_order_items")
}

enum TransferStatus {
  PENDING
  IN_TRANSIT
  RECEIVED
  CANCELLED
}

model StockTransfer {
  id          String         @id @default(cuid())
  fromStoreId String         @map("from_store_id")
  toStoreId   String         @map("to_store_id")
  status      TransferStatus @default(PENDING)
  notes       String?
  createdById String         @map("created_by_id")
  createdAt   DateTime       @default(now()) @map("created_at")

  fromStore   Store          @relation("TransferFrom", fields: [fromStoreId], references: [id])
  toStore     Store          @relation("TransferTo", fields: [toStoreId], references: [id])
  createdBy   User           @relation(fields: [createdById], references: [id])
  items       StockTransferItem[]

  @@map("stock_transfers")
}

model StockTransferItem {
  id         String        @id @default(cuid())
  transferId String        @map("transfer_id")
  productId  String        @map("product_id")
  qty        Int

  transfer   StockTransfer @relation(fields: [transferId], references: [id])
  product    Product       @relation(fields: [productId], references: [id])

  @@map("stock_transfer_items")
}

enum StockCountStatus {
  IN_PROGRESS
  SUBMITTED
  RECONCILED
}

model StockCount {
  id          String           @id @default(cuid())
  storeId     String           @map("store_id")
  status      StockCountStatus @default(IN_PROGRESS)
  notes       String?
  createdById String           @map("created_by_id")
  createdAt   DateTime         @default(now()) @map("created_at")

  store       Store            @relation(fields: [storeId], references: [id])
  createdBy   User             @relation(fields: [createdById], references: [id])
  items       StockCountItem[]

  @@map("stock_counts")
}

model StockCountItem {
  id            String     @id @default(cuid())
  stockCountId  String     @map("stock_count_id")
  productId     String     @map("product_id")
  systemQty     Int        @map("system_qty")
  countedQty    Int        @map("counted_qty")
  variance      Int

  stockCount    StockCount @relation(fields: [stockCountId], references: [id])
  product       Product    @relation(fields: [productId], references: [id])

  @@map("stock_count_items")
}
```

- [ ] **Step 2: Run migration**

```bash
cd apps/api && npx prisma migrate dev --name add_advanced_inventory
```

Expected: Migration applied successfully.

- [ ] **Step 3: Commit**

```bash
git add apps/api/prisma/
git commit -m "feat: add advanced inventory tables to Prisma schema"
```

---

## Task 2: Suppliers module

- [ ] **Step 1: Create suppliers.service.ts**

```typescript
// apps/api/src/suppliers/suppliers.service.ts
import { Injectable, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class SuppliersService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.supplier.findMany({ orderBy: { name: 'asc' } })
  }

  async findOne(id: string) {
    const s = await this.prisma.supplier.findUnique({ where: { id } })
    if (!s) throw new NotFoundException('Supplier not found')
    return s
  }

  create(dto: { name: string; contactName?: string; email?: string; phone?: string; address?: string }) {
    return this.prisma.supplier.create({ data: dto })
  }

  async update(id: string, dto: Partial<{ name: string; contactName: string; email: string; phone: string; address: string }>) {
    await this.findOne(id)
    return this.prisma.supplier.update({ where: { id }, data: dto })
  }

  async delete(id: string) {
    await this.findOne(id)
    return this.prisma.supplier.delete({ where: { id } })
  }
}
```

- [ ] **Step 2: Create suppliers.controller.ts**

```typescript
// apps/api/src/suppliers/suppliers.controller.ts
import { Controller, Get, Post, Patch, Delete, Param, Body, UseGuards } from '@nestjs/common'
import { SuppliersService } from './suppliers.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('suppliers')
@UseGuards(JwtAuthGuard, RolesGuard)
export class SuppliersController {
  constructor(private readonly service: SuppliersService) {}

  @Get() @Roles('INVENTORY') findAll() { return this.service.findAll() }
  @Post() @Roles('INVENTORY') create(@Body() body: any) { return this.service.create(body) }
  @Patch(':id') @Roles('INVENTORY') update(@Param('id') id: string, @Body() body: any) { return this.service.update(id, body) }
  @Delete(':id') @Roles('MANAGER') delete(@Param('id') id: string) { return this.service.delete(id) }
}
```

- [ ] **Step 3: Create suppliers.module.ts and register in AppModule**

```typescript
// apps/api/src/suppliers/suppliers.module.ts
import { Module } from '@nestjs/common'
import { SuppliersController } from './suppliers.controller'
import { SuppliersService } from './suppliers.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [SuppliersController],
  providers: [SuppliersService],
})
export class SuppliersModule {}
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/suppliers/
git commit -m "feat: add suppliers module"
```

---

## Task 3: Purchase orders module — TDD

- [ ] **Step 1: Write failing tests**

```typescript
// apps/api/src/purchase-orders/purchase-orders.service.spec.ts
import { Test } from '@nestjs/testing'
import { PurchaseOrdersService } from './purchase-orders.service'
import { PrismaService } from '../prisma/prisma.service'
import { NotFoundException } from '@nestjs/common'

const mockPrisma = {
  $transaction: jest.fn(),
  purchaseOrder: {
    create: jest.fn(),
    findFirst: jest.fn(),
    findMany: jest.fn(),
    update: jest.fn(),
  },
}

describe('PurchaseOrdersService', () => {
  let service: PurchaseOrdersService

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        PurchaseOrdersService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile()
    service = module.get(PurchaseOrdersService)
    jest.clearAllMocks()
  })

  describe('receiveStock', () => {
    it('throws NotFoundException if PO not found', async () => {
      mockPrisma.purchaseOrder.findFirst.mockResolvedValue(null)
      await expect(
        service.receiveStock('po1', 'store1', 'user1', []),
      ).rejects.toThrow(NotFoundException)
    })

    it('increments stock and creates movements atomically', async () => {
      const po = {
        id: 'po1',
        storeId: 'store1',
        status: 'ORDERED',
        items: [{ id: 'poi1', productId: 'p1', qty: 10, receivedQty: 0 }],
      }
      mockPrisma.purchaseOrder.findFirst.mockResolvedValue(po)
      mockPrisma.$transaction.mockImplementation(async (fn: any) => {
        const tx = {
          purchaseOrderItem: { update: jest.fn() },
          storeProduct: {
            upsert: jest.fn().mockResolvedValue({ stockQty: 10 }),
          },
          stockMovement: { create: jest.fn() },
          purchaseOrder: { update: jest.fn().mockResolvedValue({ ...po, status: 'RECEIVED' }) },
        }
        return fn(tx)
      })
      const result = await service.receiveStock('po1', 'store1', 'user1', [
        { purchaseOrderItemId: 'poi1', qty: 10 },
      ])
      expect(mockPrisma.$transaction).toHaveBeenCalledTimes(1)
    })
  })
})
```

- [ ] **Step 2: Run test to verify failure**

```bash
cd apps/api && npx jest purchase-orders.service.spec --no-coverage
```

Expected: FAIL — `PurchaseOrdersService` not found.

- [ ] **Step 3: Implement purchase-orders.service.ts**

```typescript
// apps/api/src/purchase-orders/purchase-orders.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

interface ReceiveItemInput { purchaseOrderItemId: string; qty: number }

@Injectable()
export class PurchaseOrdersService {
  constructor(private readonly prisma: PrismaService) {}

  create(
    storeId: string,
    createdById: string,
    dto: { supplierId: string; notes?: string; items: { productId: string; qty: number; costPrice: number }[] },
  ) {
    const totalAmount = dto.items.reduce((s, i) => s + i.qty * i.costPrice, 0)
    return this.prisma.purchaseOrder.create({
      data: {
        storeId,
        supplierId: dto.supplierId,
        createdById,
        notes: dto.notes ?? null,
        totalAmount,
        items: { create: dto.items },
      },
      include: { items: true, supplier: true },
    })
  }

  findAll(storeId: string) {
    return this.prisma.purchaseOrder.findMany({
      where: { storeId },
      include: { supplier: true, items: { include: { product: { select: { name: true } } } } },
      orderBy: { createdAt: 'desc' },
    })
  }

  async receiveStock(
    poId: string,
    storeId: string,
    receivedById: string,
    items: ReceiveItemInput[],
  ) {
    const po = await this.prisma.purchaseOrder.findFirst({
      where: { id: poId, storeId, status: { in: ['ORDERED', 'PARTIAL'] } },
      include: { items: true },
    })
    if (!po) throw new NotFoundException('Purchase order not found or not in receivable state')

    return this.prisma.$transaction(async (tx) => {
      for (const receive of items) {
        const poItem = po.items.find((i) => i.id === receive.purchaseOrderItemId)
        if (!poItem) throw new BadRequestException(`PO item ${receive.purchaseOrderItemId} not found`)

        const remaining = poItem.qty - poItem.receivedQty
        if (receive.qty > remaining) {
          throw new BadRequestException(`Cannot receive ${receive.qty} — only ${remaining} remaining`)
        }

        await tx.purchaseOrderItem.update({
          where: { id: poItem.id },
          data: { receivedQty: { increment: receive.qty } },
        })

        await tx.storeProduct.upsert({
          where: { storeId_productId: { storeId, productId: poItem.productId } },
          create: { storeId, productId: poItem.productId, stockQty: receive.qty },
          update: { stockQty: { increment: receive.qty } },
        })

        await tx.stockMovement.create({
          data: {
            storeId,
            productId: poItem.productId,
            type: 'RECEIVE',
            qtyChange: receive.qty,
            createdById: receivedById,
            referenceId: poId,
          },
        })
      }

      // Determine new PO status
      const updatedItems = await tx.purchaseOrderItem.findMany({ where: { purchaseOrderId: poId } })
      const allReceived = updatedItems.every((i) => i.receivedQty >= i.qty)
      const anyReceived = updatedItems.some((i) => i.receivedQty > 0)

      return tx.purchaseOrder.update({
        where: { id: poId },
        data: { status: allReceived ? 'RECEIVED' : anyReceived ? 'PARTIAL' : 'ORDERED' },
        include: { items: true },
      })
    })
  }
}
```

- [ ] **Step 4: Create purchase-orders.controller.ts**

```typescript
// apps/api/src/purchase-orders/purchase-orders.controller.ts
import { Controller, Get, Post, Patch, Param, Body, Request, UseGuards } from '@nestjs/common'
import { PurchaseOrdersService } from './purchase-orders.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('purchase-orders')
@UseGuards(JwtAuthGuard, RolesGuard)
export class PurchaseOrdersController {
  constructor(private readonly service: PurchaseOrdersService) {}

  @Get() @Roles('INVENTORY')
  findAll(@Request() req: any) { return this.service.findAll(req.user.storeId) }

  @Post() @Roles('INVENTORY')
  create(@Body() body: any, @Request() req: any) {
    return this.service.create(req.user.storeId, req.user.sub, body)
  }

  @Patch(':id/receive') @Roles('INVENTORY')
  receive(@Param('id') id: string, @Body() body: any, @Request() req: any) {
    return this.service.receiveStock(id, req.user.storeId, req.user.sub, body.items)
  }
}
```

- [ ] **Step 5: Create purchase-orders.module.ts and register in AppModule**

```typescript
// apps/api/src/purchase-orders/purchase-orders.module.ts
import { Module } from '@nestjs/common'
import { PurchaseOrdersController } from './purchase-orders.controller'
import { PurchaseOrdersService } from './purchase-orders.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [PurchaseOrdersController],
  providers: [PurchaseOrdersService],
})
export class PurchaseOrdersModule {}
```

- [ ] **Step 6: Run tests**

```bash
cd apps/api && npx jest purchase-orders.service.spec --no-coverage
```

Expected: PASS (2 tests).

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/purchase-orders/
git commit -m "feat: add purchase orders module with atomic stock receiving"
```

---

## Task 4: Stock transfers module — TDD

- [ ] **Step 1: Write failing tests**

```typescript
// apps/api/src/transfers/transfers.service.spec.ts
import { Test } from '@nestjs/testing'
import { TransfersService } from './transfers.service'
import { PrismaService } from '../prisma/prisma.service'
import { BadRequestException, NotFoundException } from '@nestjs/common'

const mockPrisma = {
  $transaction: jest.fn(),
  stockTransfer: { findUnique: jest.fn(), create: jest.fn() },
  storeProduct: { findUnique: jest.fn() },
}

describe('TransfersService', () => {
  let service: TransfersService

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        TransfersService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile()
    service = module.get(TransfersService)
    jest.clearAllMocks()
  })

  describe('createTransfer', () => {
    it('throws BadRequestException if from and to store are the same', async () => {
      await expect(
        service.createTransfer('store1', 'store1', 'user1', []),
      ).rejects.toThrow(BadRequestException)
    })
  })

  describe('receiveTransfer', () => {
    it('throws NotFoundException if transfer not found', async () => {
      mockPrisma.stockTransfer.findUnique.mockResolvedValue(null)
      await expect(service.receiveTransfer('t1', 'store2', 'user1')).rejects.toThrow(NotFoundException)
    })

    it('deducts from source and increments destination atomically', async () => {
      const transfer = {
        id: 't1',
        fromStoreId: 'store1',
        toStoreId: 'store2',
        status: 'IN_TRANSIT',
        items: [{ productId: 'p1', qty: 5 }],
      }
      mockPrisma.stockTransfer.findUnique.mockResolvedValue(transfer)
      mockPrisma.$transaction.mockImplementation(async (fn: any) => {
        const tx = {
          storeProduct: {
            update: jest.fn(),
            upsert: jest.fn(),
          },
          stockMovement: { create: jest.fn() },
          stockTransfer: { update: jest.fn().mockResolvedValue({ ...transfer, status: 'RECEIVED' }) },
        }
        return fn(tx)
      })
      await service.receiveTransfer('t1', 'store2', 'user1')
      expect(mockPrisma.$transaction).toHaveBeenCalledTimes(1)
    })
  })
})
```

- [ ] **Step 2: Run test to verify failure**

```bash
cd apps/api && npx jest transfers.service.spec --no-coverage
```

Expected: FAIL.

- [ ] **Step 3: Implement transfers.service.ts**

```typescript
// apps/api/src/transfers/transfers.service.ts
import { Injectable, BadRequestException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class TransfersService {
  constructor(private readonly prisma: PrismaService) {}

  async createTransfer(
    fromStoreId: string,
    toStoreId: string,
    createdById: string,
    items: { productId: string; qty: number }[],
    notes?: string,
  ) {
    if (fromStoreId === toStoreId) throw new BadRequestException('Cannot transfer to the same store')

    return this.prisma.stockTransfer.create({
      data: {
        fromStoreId,
        toStoreId,
        createdById,
        notes: notes ?? null,
        status: 'PENDING',
        items: { create: items },
      },
      include: { items: true },
    })
  }

  async dispatchTransfer(transferId: string, fromStoreId: string, userId: string) {
    const transfer = await this.prisma.stockTransfer.findUnique({
      where: { id: transferId },
      include: { items: true },
    })
    if (!transfer || transfer.fromStoreId !== fromStoreId || transfer.status !== 'PENDING') {
      throw new NotFoundException('Transfer not found or not in pending state')
    }

    return this.prisma.$transaction(async (tx) => {
      for (const item of transfer.items) {
        await tx.storeProduct.update({
          where: { storeId_productId: { storeId: fromStoreId, productId: item.productId } },
          data: { stockQty: { decrement: item.qty } },
        })
        await tx.stockMovement.create({
          data: {
            storeId: fromStoreId,
            productId: item.productId,
            type: 'TRANSFER_OUT',
            qtyChange: -item.qty,
            createdById: userId,
            referenceId: transferId,
          },
        })
      }
      return tx.stockTransfer.update({
        where: { id: transferId },
        data: { status: 'IN_TRANSIT' },
      })
    })
  }

  async receiveTransfer(transferId: string, toStoreId: string, userId: string) {
    const transfer = await this.prisma.stockTransfer.findUnique({
      where: { id: transferId },
      include: { items: true },
    })
    if (!transfer || transfer.toStoreId !== toStoreId || transfer.status !== 'IN_TRANSIT') {
      throw new NotFoundException('Transfer not found or not in transit')
    }

    return this.prisma.$transaction(async (tx) => {
      for (const item of transfer.items) {
        await tx.storeProduct.upsert({
          where: { storeId_productId: { storeId: toStoreId, productId: item.productId } },
          create: { storeId: toStoreId, productId: item.productId, stockQty: item.qty },
          update: { stockQty: { increment: item.qty } },
        })
        await tx.stockMovement.create({
          data: {
            storeId: toStoreId,
            productId: item.productId,
            type: 'TRANSFER_IN',
            qtyChange: item.qty,
            createdById: userId,
            referenceId: transferId,
          },
        })
      }
      return tx.stockTransfer.update({
        where: { id: transferId },
        data: { status: 'RECEIVED' },
        include: { items: true },
      })
    })
  }

  findAll(storeId: string) {
    return this.prisma.stockTransfer.findMany({
      where: { OR: [{ fromStoreId: storeId }, { toStoreId: storeId }] },
      include: {
        items: { include: { product: { select: { name: true } } } },
        fromStore: { select: { id: true, name: true } },
        toStore: { select: { id: true, name: true } },
      },
      orderBy: { createdAt: 'desc' },
    })
  }
}
```

- [ ] **Step 4: Create transfers.controller.ts**

```typescript
// apps/api/src/transfers/transfers.controller.ts
import { Controller, Get, Post, Patch, Param, Body, Request, UseGuards } from '@nestjs/common'
import { TransfersService } from './transfers.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('transfers')
@UseGuards(JwtAuthGuard, RolesGuard)
export class TransfersController {
  constructor(private readonly service: TransfersService) {}

  @Get() @Roles('INVENTORY')
  findAll(@Request() req: any) { return this.service.findAll(req.user.storeId) }

  @Post() @Roles('MANAGER')
  create(@Body() body: any, @Request() req: any) {
    return this.service.createTransfer(
      req.user.storeId, body.toStoreId, req.user.sub, body.items, body.notes,
    )
  }

  @Patch(':id/dispatch') @Roles('INVENTORY')
  dispatch(@Param('id') id: string, @Request() req: any) {
    return this.service.dispatchTransfer(id, req.user.storeId, req.user.sub)
  }

  @Patch(':id/receive') @Roles('INVENTORY')
  receive(@Param('id') id: string, @Request() req: any) {
    return this.service.receiveTransfer(id, req.user.storeId, req.user.sub)
  }
}
```

- [ ] **Step 5: Create transfers.module.ts and register in AppModule**

```typescript
// apps/api/src/transfers/transfers.module.ts
import { Module } from '@nestjs/common'
import { TransfersController } from './transfers.controller'
import { TransfersService } from './transfers.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [TransfersController],
  providers: [TransfersService],
})
export class TransfersModule {}
```

- [ ] **Step 6: Run tests**

```bash
cd apps/api && npx jest transfers.service.spec --no-coverage
```

Expected: PASS (3 tests).

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/transfers/
git commit -m "feat: add stock transfers module (dispatch + receive atomically)"
```

---

## Task 5: Low-stock BullMQ worker

- [ ] **Step 1: Install BullMQ**

```bash
pnpm add @nestjs/bullmq bullmq --filter=api
```

- [ ] **Step 2: Create low-stock.service.ts**

```typescript
// apps/api/src/low-stock/low-stock.service.ts
import { Injectable } from '@nestjs/common'
import { InjectQueue } from '@nestjs/bullmq'
import { Queue } from 'bullmq'

@Injectable()
export class LowStockService {
  constructor(@InjectQueue('low-stock') private readonly queue: Queue) {}

  async checkAfterDeduction(storeId: string, productId: string) {
    await this.queue.add('check', { storeId, productId }, { removeOnComplete: true })
  }
}
```

- [ ] **Step 3: Create low-stock.processor.ts**

```typescript
// apps/api/src/low-stock/low-stock.processor.ts
import { Processor, WorkerHost } from '@nestjs/bullmq'
import { Job } from 'bullmq'
import { Injectable } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { EventsGateway } from '../events/events.gateway'

@Injectable()
@Processor('low-stock')
export class LowStockProcessor extends WorkerHost {
  constructor(
    private readonly prisma: PrismaService,
    private readonly eventsGateway: EventsGateway,
  ) {
    super()
  }

  async process(job: Job<{ storeId: string; productId: string }>) {
    const { storeId, productId } = job.data

    const sp = await this.prisma.storeProduct.findUnique({
      where: { storeId_productId: { storeId, productId } },
      include: { product: { select: { name: true } } },
    })

    if (!sp) return

    if (sp.minStockQty != null && sp.stockQty <= sp.minStockQty) {
      this.eventsGateway.emitToStore(storeId, 'stock:low', {
        productId,
        storeId,
        qty: sp.stockQty,
        minQty: sp.minStockQty,
        productName: sp.product.name,
      })
    }
  }
}
```

- [ ] **Step 4: Create low-stock.module.ts**

```typescript
// apps/api/src/low-stock/low-stock.module.ts
import { Module } from '@nestjs/common'
import { BullModule } from '@nestjs/bullmq'
import { LowStockService } from './low-stock.service'
import { LowStockProcessor } from './low-stock.processor'
import { PrismaModule } from '../prisma/prisma.module'
import { EventsModule } from '../events/events.module'

@Module({
  imports: [
    BullModule.registerQueue({ name: 'low-stock' }),
    PrismaModule,
    EventsModule,
  ],
  providers: [LowStockService, LowStockProcessor],
  exports: [LowStockService],
})
export class LowStockModule {}
```

- [ ] **Step 5: Register BullMQ root in AppModule**

In `app.module.ts`, add to imports:

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (config: ConfigService) => ({
    connection: { host: config.get('REDIS_HOST', 'localhost'), port: config.get('REDIS_PORT', 6379) },
  }),
  inject: [ConfigService],
})
```

Import `LowStockModule` in `app.module.ts`.

- [ ] **Step 6: Wire LowStockService into OrdersModule**

In `orders.service.ts`, inject `LowStockService` and call `checkAfterDeduction` for each item after a sale:

```typescript
// In orders.module.ts — add LowStockModule to imports
// In orders.service.ts constructor:
constructor(
  private readonly prisma: PrismaService,
  private readonly eventsGateway: EventsGateway,
  private readonly lowStockService: LowStockService,
) {}

// After the stock:updated emit loop in createOrder():
for (const item of dto.items) {
  await this.lowStockService.checkAfterDeduction(storeId, item.productId)
}
```

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/low-stock/ apps/api/src/orders/
git commit -m "feat: add BullMQ low-stock checker with WebSocket alert emission"
```

---

## Task 6: Stock count module

- [ ] **Step 1: Create stock-count.service.ts**

```typescript
// apps/api/src/stock-count/stock-count.service.ts
import { Injectable, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class StockCountService {
  constructor(private readonly prisma: PrismaService) {}

  async startCount(storeId: string, createdById: string) {
    // Snapshot all current stock quantities
    const storeProducts = await this.prisma.storeProduct.findMany({
      where: { storeId },
      select: { productId: true, stockQty: true },
    })

    return this.prisma.stockCount.create({
      data: {
        storeId,
        createdById,
        status: 'IN_PROGRESS',
        items: {
          create: storeProducts.map((sp) => ({
            productId: sp.productId,
            systemQty: sp.stockQty,
            countedQty: sp.stockQty, // default to system qty — cashier updates this
            variance: 0,
          })),
        },
      },
      include: { items: { include: { product: { select: { name: true, sku: true } } } } },
    })
  }

  async updateCountItem(
    stockCountId: string,
    productId: string,
    countedQty: number,
  ) {
    const item = await this.prisma.stockCountItem.findFirst({
      where: { stockCountId, productId },
    })
    if (!item) throw new NotFoundException('Count item not found')

    return this.prisma.stockCountItem.update({
      where: { id: item.id },
      data: { countedQty, variance: countedQty - item.systemQty },
    })
  }

  async reconcile(stockCountId: string, storeId: string, userId: string) {
    const count = await this.prisma.stockCount.findFirst({
      where: { id: stockCountId, storeId, status: 'SUBMITTED' },
      include: { items: true },
    })
    if (!count) throw new NotFoundException('Stock count not found or not submitted')

    return this.prisma.$transaction(async (tx) => {
      for (const item of count.items) {
        if (item.variance === 0) continue

        await tx.storeProduct.update({
          where: { storeId_productId: { storeId, productId: item.productId } },
          data: { stockQty: item.countedQty },
        })

        await tx.stockMovement.create({
          data: {
            storeId,
            productId: item.productId,
            type: 'ADJUST',
            qtyChange: item.variance,
            createdById: userId,
            referenceId: stockCountId,
            note: `Stock count reconciliation`,
          },
        })
      }

      return tx.stockCount.update({
        where: { id: stockCountId },
        data: { status: 'RECONCILED' },
      })
    })
  }

  findAll(storeId: string) {
    return this.prisma.stockCount.findMany({
      where: { storeId },
      include: { createdBy: { select: { name: true } } },
      orderBy: { createdAt: 'desc' },
    })
  }
}
```

- [ ] **Step 2: Create stock-count.controller.ts**

```typescript
// apps/api/src/stock-count/stock-count.controller.ts
import { Controller, Get, Post, Patch, Param, Body, Request, UseGuards } from '@nestjs/common'
import { StockCountService } from './stock-count.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('stock-counts')
@UseGuards(JwtAuthGuard, RolesGuard)
export class StockCountController {
  constructor(private readonly service: StockCountService) {}

  @Get() @Roles('INVENTORY')
  findAll(@Request() req: any) { return this.service.findAll(req.user.storeId) }

  @Post() @Roles('INVENTORY')
  start(@Request() req: any) { return this.service.startCount(req.user.storeId, req.user.sub) }

  @Patch(':id/items/:productId') @Roles('INVENTORY')
  updateItem(
    @Param('id') id: string,
    @Param('productId') productId: string,
    @Body() body: { countedQty: number },
  ) {
    return this.service.updateCountItem(id, productId, body.countedQty)
  }

  @Patch(':id/reconcile') @Roles('MANAGER')
  reconcile(@Param('id') id: string, @Request() req: any) {
    return this.service.reconcile(id, req.user.storeId, req.user.sub)
  }
}
```

- [ ] **Step 3: Create stock-count.module.ts and register in AppModule**

```typescript
// apps/api/src/stock-count/stock-count.module.ts
import { Module } from '@nestjs/common'
import { StockCountController } from './stock-count.controller'
import { StockCountService } from './stock-count.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [StockCountController],
  providers: [StockCountService],
})
export class StockCountModule {}
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/stock-count/
git commit -m "feat: add stock count module with reconciliation"
```

---

## Task 7: Frontend — stock panel advanced pages

- [ ] **Step 1: Create purchase orders page**

```tsx
// apps/web/src/app/(stock)/stock/purchase-orders/page.tsx
'use client'
import { useQuery } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'

const STATUS_COLORS: Record<string, string> = {
  DRAFT: 'bg-canvas text-text-muted border border-border',
  ORDERED: 'bg-info/10 text-info',
  PARTIAL: 'bg-warning/10 text-warning',
  RECEIVED: 'bg-primary/10 text-primary',
  CANCELLED: 'bg-danger/10 text-danger',
}

export default function PurchaseOrdersPage() {
  const { data } = useQuery({
    queryKey: ['purchase-orders'],
    queryFn: () => apiFetch('/purchase-orders').then((r) => r.data),
  })

  return (
    <div className="p-6">
      <h1 className="text-lg font-bold text-text-dark mb-6">Purchase Orders</h1>
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['ID', 'Supplier', 'Items', 'Total', 'Status'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">{h}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(data ?? []).map((po: any) => (
              <tr key={po.id} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 font-mono text-xs text-text-muted">{po.id.slice(-6)}</td>
                <td className="px-4 py-3 font-semibold">{po.supplier?.name}</td>
                <td className="px-4 py-3 text-text-muted">{po.items?.length}</td>
                <td className="px-4 py-3 font-bold">{po.totalAmount.toLocaleString()} Ks</td>
                <td className="px-4 py-3">
                  <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${STATUS_COLORS[po.status]}`}>
                    {po.status}
                  </span>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create transfers page**

```tsx
// apps/web/src/app/(stock)/stock/transfers/page.tsx
'use client'
import { useQuery } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'

const STATUS_COLORS: Record<string, string> = {
  PENDING: 'bg-canvas text-text-muted border border-border',
  IN_TRANSIT: 'bg-info/10 text-info',
  RECEIVED: 'bg-primary/10 text-primary',
  CANCELLED: 'bg-danger/10 text-danger',
}

export default function TransfersPage() {
  const { data } = useQuery({
    queryKey: ['transfers'],
    queryFn: () => apiFetch('/transfers').then((r) => r.data),
  })

  return (
    <div className="p-6">
      <h1 className="text-lg font-bold text-text-dark mb-6">Stock Transfers</h1>
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['From', 'To', 'Items', 'Status', 'Date'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">{h}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(data ?? []).map((t: any) => (
              <tr key={t.id} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 font-semibold">{t.fromStore?.name}</td>
                <td className="px-4 py-3 font-semibold">{t.toStore?.name}</td>
                <td className="px-4 py-3 text-text-muted">{t.items?.length}</td>
                <td className="px-4 py-3">
                  <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${STATUS_COLORS[t.status]}`}>
                    {t.status}
                  </span>
                </td>
                <td className="px-4 py-3 text-text-muted">{new Date(t.createdAt).toLocaleDateString()}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create low stock page**

```tsx
// apps/web/src/app/(stock)/stock/low-stock/page.tsx
'use client'
import { useEffect, useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'
import { getSocket } from '@/lib/socket'
import { useAuthStore } from '@/lib/auth-store'
import { AlertTriangle } from 'lucide-react'

export default function LowStockPage() {
  const { token } = useAuthStore()
  const [liveAlerts, setLiveAlerts] = useState<any[]>([])

  const { data: lowStockItems } = useQuery({
    queryKey: ['low-stock'],
    queryFn: () =>
      apiFetch('/stores/current/stock?lowStockOnly=true').then((r) => r.data),
    staleTime: 30_000,
  })

  useEffect(() => {
    if (!token) return
    const socket = getSocket(token)
    socket.on('stock:low', (payload: any) => {
      setLiveAlerts((prev) => [{ ...payload, at: new Date() }, ...prev].slice(0, 20))
    })
    return () => { socket.off('stock:low') }
  }, [token])

  return (
    <div className="p-6 space-y-6">
      <h1 className="text-lg font-bold text-text-dark">Low Stock Alerts</h1>

      {liveAlerts.length > 0 && (
        <div className="space-y-2">
          <p className="text-xs font-bold uppercase tracking-wide text-warning">Live Alerts</p>
          {liveAlerts.map((a, i) => (
            <div key={i} className="flex items-center gap-3 p-3 bg-warning/10 border border-warning/30 rounded-xl">
              <AlertTriangle size={16} className="text-warning flex-shrink-0" />
              <div>
                <p className="text-sm font-semibold">{a.productName}</p>
                <p className="text-xs text-text-muted">
                  Stock: {a.qty} (min: {a.minQty}) · {new Date(a.at).toLocaleTimeString()}
                </p>
              </div>
            </div>
          ))}
        </div>
      )}

      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['Product', 'Current Stock', 'Min Stock', 'Variance'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">{h}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(lowStockItems ?? []).map((item: any) => (
              <tr key={item.productId} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 font-semibold">{item.product?.name}</td>
                <td className="px-4 py-3 font-bold text-danger">{item.stockQty}</td>
                <td className="px-4 py-3 text-text-muted">{item.minStockQty}</td>
                <td className="px-4 py-3 text-danger font-semibold">
                  {item.stockQty - item.minStockQty}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create stock count page**

```tsx
// apps/web/src/app/(stock)/stock/count/page.tsx
'use client'
import { useState } from 'react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'

export default function StockCountPage() {
  const qc = useQueryClient()
  const [activeCountId, setActiveCountId] = useState<string | null>(null)

  const { data: counts } = useQuery({
    queryKey: ['stock-counts'],
    queryFn: () => apiFetch('/stock-counts').then((r) => r.data),
  })

  const { data: activeCount } = useQuery({
    queryKey: ['stock-count', activeCountId],
    queryFn: () => apiFetch(`/stock-counts/${activeCountId}`).then((r) => r.data),
    enabled: !!activeCountId,
  })

  const startMutation = useMutation({
    mutationFn: () => apiFetch('/stock-counts', { method: 'POST' }).then((r) => r.data),
    onSuccess: (data) => { setActiveCountId(data.id); qc.invalidateQueries({ queryKey: ['stock-counts'] }) },
  })

  const updateMutation = useMutation({
    mutationFn: ({ productId, countedQty }: { productId: string; countedQty: number }) =>
      apiFetch(`/stock-counts/${activeCountId}/items/${productId}`, {
        method: 'PATCH',
        body: JSON.stringify({ countedQty }),
      }),
  })

  return (
    <div className="p-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-lg font-bold text-text-dark">Stock Count</h1>
        {!activeCountId && (
          <button
            onClick={() => startMutation.mutate()}
            disabled={startMutation.isPending}
            className="px-4 py-2 bg-primary text-white rounded-xl text-sm font-semibold min-h-[44px] disabled:opacity-40"
          >
            {startMutation.isPending ? 'Starting…' : 'Start New Count'}
          </button>
        )}
      </div>

      {activeCount && (
        <div className="bg-surface border border-border rounded-xl overflow-hidden">
          <div className="px-4 py-3 bg-canvas border-b border-border">
            <p className="text-xs font-bold uppercase tracking-wide text-text-muted">
              Active Count — {activeCount.items?.length} products
            </p>
          </div>
          <div className="divide-y divide-border">
            {(activeCount.items ?? []).map((item: any) => (
              <div key={item.id} className="flex items-center gap-4 px-4 py-3">
                <div className="flex-1">
                  <p className="text-sm font-semibold">{item.product?.name}</p>
                  <p className="text-xs text-text-muted">System: {item.systemQty}</p>
                </div>
                <input
                  type="number"
                  defaultValue={item.countedQty}
                  onBlur={(e) =>
                    updateMutation.mutate({
                      productId: item.productId,
                      countedQty: parseInt(e.target.value),
                    })
                  }
                  className="w-24 px-3 py-2 border border-border rounded-lg text-sm text-center focus:outline-none focus:border-primary min-h-[44px]"
                  style={{ fontSize: '16px' }}
                />
                {item.variance !== 0 && (
                  <span className={`text-xs font-bold min-w-[40px] text-right ${item.variance > 0 ? 'text-primary' : 'text-danger'}`}>
                    {item.variance > 0 ? '+' : ''}{item.variance}
                  </span>
                )}
              </div>
            ))}
          </div>
        </div>
      )}

      {!activeCountId && counts?.length > 0 && (
        <div className="space-y-2">
          <p className="text-xs font-bold uppercase tracking-wide text-text-muted">Previous Counts</p>
          {counts.map((c: any) => (
            <div key={c.id} className="bg-surface border border-border rounded-xl p-4 flex items-center justify-between">
              <div>
                <p className="text-sm font-semibold">{new Date(c.createdAt).toLocaleDateString()}</p>
                <p className="text-xs text-text-muted">By {c.createdBy?.name}</p>
              </div>
              <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${
                c.status === 'RECONCILED' ? 'bg-primary/10 text-primary' :
                c.status === 'SUBMITTED' ? 'bg-info/10 text-info' :
                'bg-canvas text-text-muted border border-border'
              }`}>{c.status}</span>
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 5: Run all tests**

```bash
cd apps/api && npx jest --no-coverage
```

Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/app/(stock)/stock/ apps/api/src/
git commit -m "feat: complete Phase 7 — purchase orders, transfers, stock count, low-stock pages"
```

---

## Task 8: Final integration check

- [ ] **Step 1: Start full stack**

```bash
pnpm dev
```

- [ ] **Step 2: Verify advanced inventory flows**

1. Navigate to `/stock/purchase-orders` — confirm table loads
2. Navigate to `/stock/transfers` — confirm table loads
3. Navigate to `/stock/count` — click Start New Count — confirm product list loads
4. Enter counted quantities and blur each field — confirm values persist
5. Navigate to `/stock/low-stock` — reduce a product below its min via adjustment — confirm WebSocket alert appears in real time

- [ ] **Step 3: Push all plans to GitHub**

```bash
git add docs/superpowers/plans/
git commit -m "docs: add Phase 3–7 implementation plans"
git push origin main
```
