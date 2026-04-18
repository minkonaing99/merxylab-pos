# Phase 3 — POS Selling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the full cashier POS experience — shift management, product grid + cart, order creation with atomic stock deduction, hold/resume, payment modal, digital receipt, and real-time WebSocket events.

**Architecture:** Orders module creates orders transactionally: each `OrderItem` deducts stock atomically via a `StockMovement` row + `StoreProduct.stockQty` update in one Prisma `$transaction`. WebSocket gateway emits `order:completed` and `stock:updated` into the store's Socket.IO room. Frontend caches the product list in IndexedDB so the cart remains usable when offline — the Pay button is disabled while disconnected.

**Tech Stack:** NestJS modules (shifts, orders), Prisma transactions, Socket.IO gateway, Next.js POS panel, Zustand cart store, React Hook Form + Zod, IndexedDB (idb library), Tailwind CSS touch rules

**Depends on:** Phase 1 complete (auth, JWT guard, design system), Phase 2 complete (products + inventory)

---

## File Map

```
apps/api/src/
├── prisma/schema.prisma                       # add Shift, Order, OrderItem
├── shifts/
│   ├── shifts.module.ts
│   ├── shifts.controller.ts
│   ├── shifts.service.ts
│   └── shifts.service.spec.ts
├── orders/
│   ├── orders.module.ts
│   ├── orders.controller.ts
│   ├── orders.service.ts
│   ├── orders.service.spec.ts
│   └── dto/
│       ├── create-order.dto.ts
│       └── hold-order.dto.ts
└── events/
    ├── events.module.ts
    └── events.gateway.ts                      # Socket.IO gateway

apps/web/src/
├── app/(pos)/
│   ├── pos/
│   │   ├── page.tsx                           # POS main screen
│   │   └── _components/
│   │       ├── ProductGrid.tsx
│   │       ├── ProductCard.tsx
│   │       ├── CategoryChips.tsx
│   │       ├── CartPanel.tsx
│   │       ├── CartItem.tsx
│   │       ├── PaymentModal.tsx
│   │       ├── ReceiptPreview.tsx
│   │       └── OfflineBanner.tsx
│   ├── pos/shift/
│   │   ├── open/page.tsx                      # Open shift screen
│   │   └── close/page.tsx                     # Close shift + summary
│   └── pos/held/page.tsx                      # Held orders list
└── lib/
    ├── cart-store.ts                           # Zustand cart store
    ├── product-cache.ts                        # IndexedDB product cache (idb)
    └── socket.ts                               # Socket.IO client singleton

packages/types/src/
├── shift.types.ts
└── order.types.ts
```

---

## Task 1: Extend Prisma schema — Shift, Order, OrderItem

- [ ] **Step 1: Add enums and models**

Append to `schema.prisma` after the `StockMovement` model:

```prisma
enum OrderStatus {
  PENDING
  COMPLETED
  REFUNDED
  HELD
  CANCELLED
}

enum PaymentMethod {
  CASH
  CARD
}

model Shift {
  id           String    @id @default(cuid())
  storeId      String    @map("store_id")
  userId       String    @map("user_id")
  openedAt     DateTime  @default(now()) @map("opened_at")
  closedAt     DateTime? @map("closed_at")
  openingCash  Int       @default(0) @map("opening_cash")
  closingCash  Int?      @map("closing_cash")
  notes        String?

  store        Store     @relation(fields: [storeId], references: [id])
  user         User      @relation(fields: [userId], references: [id])
  orders       Order[]

  @@map("shifts")
}

model Order {
  id            String        @id @default(cuid())
  storeId       String        @map("store_id")
  shiftId       String        @map("shift_id")
  cashierId     String        @map("cashier_id")
  orderNumber   Int           @map("order_number")
  status        OrderStatus   @default(PENDING)
  subtotal      Int
  discountAmount Int          @default(0) @map("discount_amount")
  taxAmount     Int           @default(0) @map("tax_amount")
  total         Int
  paymentMethod PaymentMethod? @map("payment_method")
  amountTendered Int?         @map("amount_tendered")
  changeAmount  Int?          @map("change_amount")
  note          String?
  createdAt     DateTime      @default(now()) @map("created_at")

  store         Store         @relation(fields: [storeId], references: [id])
  shift         Shift         @relation(fields: [shiftId], references: [id])
  cashier       User          @relation(fields: [cashierId], references: [id])
  items         OrderItem[]
  refunds       Refund[]

  @@unique([storeId, orderNumber])
  @@map("orders")
}

model OrderItem {
  id          String  @id @default(cuid())
  orderId     String  @map("order_id")
  productId   String  @map("product_id")
  productName String  @map("product_name")
  qty         Int
  unitPrice   Int     @map("unit_price")
  discount    Int     @default(0)
  total       Int

  order       Order   @relation(fields: [orderId], references: [id])
  product     Product @relation(fields: [productId], references: [id])

  @@map("order_items")
}
```

- [ ] **Step 2: Run migration**

```bash
cd apps/api
npx prisma migrate dev --name add_shifts_orders
```

Expected: Migration created and applied successfully.

- [ ] **Step 3: Commit**

```bash
git add apps/api/prisma/
git commit -m "feat: add Shift, Order, OrderItem to Prisma schema"
```

---

## Task 2: Shared types — Shift and Order

- [ ] **Step 1: Create shift.types.ts**

```typescript
// packages/types/src/shift.types.ts
export interface ShiftDto {
  id: string
  storeId: string
  userId: string
  openedAt: string
  closedAt: string | null
  openingCash: number
  closingCash: number | null
  notes: string | null
  orderCount?: number
  totalSales?: number
}

export interface OpenShiftDto {
  openingCash: number
  notes?: string
}

export interface CloseShiftDto {
  closingCash: number
  notes?: string
}
```

- [ ] **Step 2: Create order.types.ts**

```typescript
// packages/types/src/order.types.ts
export type OrderStatus = 'PENDING' | 'COMPLETED' | 'REFUNDED' | 'HELD' | 'CANCELLED'
export type PaymentMethod = 'CASH' | 'CARD'

export interface OrderItemInput {
  productId: string
  productName: string
  qty: number
  unitPrice: number
  discount: number
}

export interface CreateOrderDto {
  items: OrderItemInput[]
  paymentMethod: PaymentMethod
  amountTendered?: number
  discountAmount?: number
  taxAmount?: number
  note?: string
}

export interface OrderItemDto {
  id: string
  productId: string
  productName: string
  qty: number
  unitPrice: number
  discount: number
  total: number
}

export interface OrderDto {
  id: string
  storeId: string
  shiftId: string
  cashierId: string
  orderNumber: number
  status: OrderStatus
  subtotal: number
  discountAmount: number
  taxAmount: number
  total: number
  paymentMethod: PaymentMethod | null
  amountTendered: number | null
  changeAmount: number | null
  note: string | null
  createdAt: string
  items: OrderItemDto[]
}
```

- [ ] **Step 3: Export from index**

```typescript
// packages/types/src/index.ts  — add these exports
export * from './shift.types'
export * from './order.types'
```

- [ ] **Step 4: Commit**

```bash
git add packages/types/src/
git commit -m "feat: add shift and order shared types"
```

---

## Task 3: Shifts module (NestJS)

- [ ] **Step 1: Write the failing test**

```typescript
// apps/api/src/shifts/shifts.service.spec.ts
import { Test } from '@nestjs/testing'
import { ShiftsService } from './shifts.service'
import { PrismaService } from '../prisma/prisma.service'
import { ConflictException, NotFoundException } from '@nestjs/common'

const mockPrisma = {
  shift: {
    findFirst: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    findMany: jest.fn(),
    aggregate: jest.fn(),
  },
}

describe('ShiftsService', () => {
  let service: ShiftsService

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        ShiftsService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile()
    service = module.get(ShiftsService)
    jest.clearAllMocks()
  })

  describe('openShift', () => {
    it('throws ConflictException if shift already open', async () => {
      mockPrisma.shift.findFirst.mockResolvedValue({ id: 'existing' })
      await expect(
        service.openShift('store1', 'user1', { openingCash: 5000 }),
      ).rejects.toThrow(ConflictException)
    })

    it('creates a new shift when none is open', async () => {
      mockPrisma.shift.findFirst.mockResolvedValue(null)
      const created = { id: 'shift1', storeId: 'store1', userId: 'user1', openingCash: 5000 }
      mockPrisma.shift.create.mockResolvedValue(created)
      const result = await service.openShift('store1', 'user1', { openingCash: 5000 })
      expect(result).toEqual(created)
    })
  })

  describe('closeShift', () => {
    it('throws NotFoundException if shift not found', async () => {
      mockPrisma.shift.findFirst.mockResolvedValue(null)
      await expect(
        service.closeShift('shift1', 'store1', 'user1', { closingCash: 10000 }),
      ).rejects.toThrow(NotFoundException)
    })

    it('closes an open shift', async () => {
      const shift = { id: 'shift1', closedAt: null }
      mockPrisma.shift.findFirst.mockResolvedValue(shift)
      const updated = { ...shift, closedAt: new Date() }
      mockPrisma.shift.update.mockResolvedValue(updated)
      const result = await service.closeShift('shift1', 'store1', 'user1', { closingCash: 10000 })
      expect(result.closedAt).toBeTruthy()
    })
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/api && npx jest shifts.service.spec --no-coverage
```

Expected: FAIL — `ShiftsService` not found.

- [ ] **Step 3: Implement shifts.service.ts**

```typescript
// apps/api/src/shifts/shifts.service.ts
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class ShiftsService {
  constructor(private readonly prisma: PrismaService) {}

  async openShift(
    storeId: string,
    userId: string,
    dto: { openingCash: number; notes?: string },
  ) {
    const existing = await this.prisma.shift.findFirst({
      where: { storeId, userId, closedAt: null },
    })
    if (existing) throw new ConflictException('Shift already open')

    return this.prisma.shift.create({
      data: { storeId, userId, openingCash: dto.openingCash, notes: dto.notes ?? null },
    })
  }

  async closeShift(
    shiftId: string,
    storeId: string,
    userId: string,
    dto: { closingCash: number; notes?: string },
  ) {
    const shift = await this.prisma.shift.findFirst({
      where: { id: shiftId, storeId, userId, closedAt: null },
    })
    if (!shift) throw new NotFoundException('Open shift not found')

    return this.prisma.shift.update({
      where: { id: shiftId },
      data: { closedAt: new Date(), closingCash: dto.closingCash, notes: dto.notes ?? null },
    })
  }

  async getActiveShift(storeId: string, userId: string) {
    return this.prisma.shift.findFirst({
      where: { storeId, userId, closedAt: null },
    })
  }

  async getShiftSummary(shiftId: string, storeId: string) {
    const shift = await this.prisma.shift.findFirst({
      where: { id: shiftId, storeId },
    })
    if (!shift) throw new NotFoundException('Shift not found')

    const agg = await this.prisma.order.aggregate({
      where: { shiftId, status: 'COMPLETED' },
      _sum: { total: true },
      _count: { id: true },
    })

    return {
      ...shift,
      orderCount: agg._count.id,
      totalSales: agg._sum.total ?? 0,
    }
  }
}
```

- [ ] **Step 4: Create shifts.controller.ts**

```typescript
// apps/api/src/shifts/shifts.controller.ts
import { Controller, Post, Patch, Get, Param, Body, Request } from '@nestjs/common'
import { ShiftsService } from './shifts.service'
import { Roles } from '../auth/roles.decorator'
import { UseGuards } from '@nestjs/common'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'

@Controller('shifts')
@UseGuards(JwtAuthGuard, RolesGuard)
export class ShiftsController {
  constructor(private readonly shiftsService: ShiftsService) {}

  @Post('open')
  @Roles('CASHIER')
  openShift(@Body() body: { openingCash: number; notes?: string }, @Request() req: any) {
    return this.shiftsService.openShift(req.user.storeId, req.user.sub, body)
  }

  @Patch(':id/close')
  @Roles('CASHIER')
  closeShift(
    @Param('id') id: string,
    @Body() body: { closingCash: number; notes?: string },
    @Request() req: any,
  ) {
    return this.shiftsService.closeShift(id, req.user.storeId, req.user.sub, body)
  }

  @Get('active')
  @Roles('CASHIER')
  getActive(@Request() req: any) {
    return this.shiftsService.getActiveShift(req.user.storeId, req.user.sub)
  }

  @Get(':id/summary')
  @Roles('CASHIER')
  getSummary(@Param('id') id: string, @Request() req: any) {
    return this.shiftsService.getShiftSummary(id, req.user.storeId)
  }
}
```

- [ ] **Step 5: Create shifts.module.ts**

```typescript
// apps/api/src/shifts/shifts.module.ts
import { Module } from '@nestjs/common'
import { ShiftsController } from './shifts.controller'
import { ShiftsService } from './shifts.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [ShiftsController],
  providers: [ShiftsService],
  exports: [ShiftsService],
})
export class ShiftsModule {}
```

- [ ] **Step 6: Register in AppModule**

In `apps/api/src/app.module.ts`, add `ShiftsModule` to the imports array.

- [ ] **Step 7: Run tests**

```bash
cd apps/api && npx jest shifts.service.spec --no-coverage
```

Expected: PASS (2 suites, 4 tests).

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/shifts/
git commit -m "feat: add shifts module (open/close/summary)"
```

---

## Task 4: WebSocket gateway (Socket.IO)

- [ ] **Step 1: Create events.gateway.ts**

```typescript
// apps/api/src/events/events.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
} from '@nestjs/websockets'
import { Server, Socket } from 'socket.io'
import { JwtService } from '@nestjs/jwt'
import { Injectable } from '@nestjs/common'

@Injectable()
@WebSocketGateway({ cors: { origin: '*' }, namespace: '/' })
export class EventsGateway implements OnGatewayConnection {
  @WebSocketServer()
  server: Server

  constructor(private readonly jwtService: JwtService) {}

  async handleConnection(client: Socket) {
    const token = client.handshake.auth?.token as string | undefined
    if (!token) { client.disconnect(); return }
    try {
      const payload = this.jwtService.verify(token)
      client.data.storeId = payload.storeId
      if (payload.storeId) {
        client.join(`store:${payload.storeId}`)
      }
    } catch {
      client.disconnect()
    }
  }

  emitToStore(storeId: string, event: string, payload: unknown) {
    this.server.to(`store:${storeId}`).emit(event, payload)
  }
}
```

- [ ] **Step 2: Create events.module.ts**

```typescript
// apps/api/src/events/events.module.ts
import { Module } from '@nestjs/common'
import { EventsGateway } from './events.gateway'
import { JwtModule } from '@nestjs/jwt'
import { ConfigModule, ConfigService } from '@nestjs/config'

@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        secret: config.get<string>('JWT_SECRET'),
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [EventsGateway],
  exports: [EventsGateway],
})
export class EventsModule {}
```

- [ ] **Step 3: Register EventsModule in AppModule**

In `apps/api/src/app.module.ts`, import `EventsModule`.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/events/
git commit -m "feat: add Socket.IO events gateway with store-scoped rooms"
```

---

## Task 5: Orders module (NestJS) — TDD

- [ ] **Step 1: Write failing tests**

```typescript
// apps/api/src/orders/orders.service.spec.ts
import { Test } from '@nestjs/testing'
import { OrdersService } from './orders.service'
import { PrismaService } from '../prisma/prisma.service'
import { EventsGateway } from '../events/events.gateway'
import { BadRequestException } from '@nestjs/common'

const mockPrisma = {
  $transaction: jest.fn(),
  order: {
    findMany: jest.fn(),
    findFirst: jest.fn(),
    aggregate: jest.fn(),
    update: jest.fn(),
  },
  storeProduct: { findUnique: jest.fn() },
  stockMovement: { create: jest.fn() },
}
const mockGateway = { emitToStore: jest.fn() }

describe('OrdersService', () => {
  let service: OrdersService

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        OrdersService,
        { provide: PrismaService, useValue: mockPrisma },
        { provide: EventsGateway, useValue: mockGateway },
      ],
    }).compile()
    service = module.get(OrdersService)
    jest.clearAllMocks()
  })

  describe('createOrder', () => {
    it('deducts stock atomically for each item', async () => {
      const dto = {
        items: [{ productId: 'p1', productName: 'Tea', qty: 2, unitPrice: 1500, discount: 0 }],
        paymentMethod: 'CASH' as const,
        amountTendered: 3000,
      }
      const sp = { id: 'sp1', stockQty: 10 }
      mockPrisma.$transaction.mockImplementation(async (fn: any) => {
        const tx = {
          storeProduct: {
            findUnique: jest.fn().mockResolvedValue(sp),
            update: jest.fn().mockResolvedValue({ ...sp, stockQty: 8 }),
          },
          stockMovement: { create: jest.fn().mockResolvedValue({}) },
          order: {
            aggregate: jest.fn().mockResolvedValue({ _max: { orderNumber: 5 } }),
            create: jest.fn().mockResolvedValue({ id: 'o1', orderNumber: 6 }),
          },
        }
        return fn(tx)
      })
      await service.createOrder('store1', 'shift1', 'cashier1', dto)
      expect(mockPrisma.$transaction).toHaveBeenCalledTimes(1)
    })

    it('throws BadRequestException if stock insufficient', async () => {
      const dto = {
        items: [{ productId: 'p1', productName: 'Tea', qty: 5, unitPrice: 1500, discount: 0 }],
        paymentMethod: 'CASH' as const,
        amountTendered: 7500,
      }
      mockPrisma.$transaction.mockImplementation(async (fn: any) => {
        const tx = {
          storeProduct: {
            findUnique: jest.fn().mockResolvedValue({ id: 'sp1', stockQty: 2 }),
            update: jest.fn(),
          },
          stockMovement: { create: jest.fn() },
          order: { aggregate: jest.fn().mockResolvedValue({ _max: { orderNumber: null } }), create: jest.fn() },
        }
        return fn(tx)
      })
      await expect(
        service.createOrder('store1', 'shift1', 'cashier1', dto),
      ).rejects.toThrow(BadRequestException)
    })
  })
})
```

- [ ] **Step 2: Run test to verify failure**

```bash
cd apps/api && npx jest orders.service.spec --no-coverage
```

Expected: FAIL — `OrdersService` not found.

- [ ] **Step 3: Implement orders.service.ts**

```typescript
// apps/api/src/orders/orders.service.ts
import { Injectable, BadRequestException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { EventsGateway } from '../events/events.gateway'

interface OrderItemInput {
  productId: string
  productName: string
  qty: number
  unitPrice: number
  discount: number
}

interface CreateOrderInput {
  items: OrderItemInput[]
  paymentMethod: 'CASH' | 'CARD'
  amountTendered?: number
  discountAmount?: number
  taxAmount?: number
  note?: string
}

@Injectable()
export class OrdersService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly eventsGateway: EventsGateway,
  ) {}

  async createOrder(
    storeId: string,
    shiftId: string,
    cashierId: string,
    dto: CreateOrderInput,
  ) {
    const order = await this.prisma.$transaction(async (tx) => {
      // Assign sequential order number
      const agg = await tx.order.aggregate({
        where: { storeId },
        _max: { orderNumber: true },
      })
      const orderNumber = (agg._max.orderNumber ?? 0) + 1

      // Deduct stock for each item atomically
      for (const item of dto.items) {
        const sp = await tx.storeProduct.findUnique({
          where: { storeId_productId: { storeId, productId: item.productId } },
        })
        if (!sp) throw new BadRequestException(`Product ${item.productId} not in this store`)

        if (sp.stockQty < item.qty) {
          throw new BadRequestException(
            `Insufficient stock for "${item.productName}": have ${sp.stockQty}, need ${item.qty}`,
          )
        }

        await tx.stockMovement.create({
          data: {
            storeId,
            productId: item.productId,
            type: 'SALE',
            qtyChange: -item.qty,
            createdById: cashierId,
          },
        })

        await tx.storeProduct.update({
          where: { storeId_productId: { storeId, productId: item.productId } },
          data: { stockQty: { decrement: item.qty } },
        })
      }

      const subtotal = dto.items.reduce(
        (sum, i) => sum + (i.unitPrice - i.discount) * i.qty,
        0,
      )
      const discountAmount = dto.discountAmount ?? 0
      const taxAmount = dto.taxAmount ?? 0
      const total = subtotal - discountAmount + taxAmount
      const changeAmount =
        dto.paymentMethod === 'CASH' && dto.amountTendered != null
          ? dto.amountTendered - total
          : null

      return tx.order.create({
        data: {
          storeId,
          shiftId,
          cashierId,
          orderNumber,
          status: 'COMPLETED',
          subtotal,
          discountAmount,
          taxAmount,
          total,
          paymentMethod: dto.paymentMethod,
          amountTendered: dto.amountTendered ?? null,
          changeAmount,
          note: dto.note ?? null,
          items: {
            create: dto.items.map((i) => ({
              productId: i.productId,
              productName: i.productName,
              qty: i.qty,
              unitPrice: i.unitPrice,
              discount: i.discount,
              total: (i.unitPrice - i.discount) * i.qty,
            })),
          },
        },
        include: { items: true },
      })
    })

    this.eventsGateway.emitToStore(storeId, 'order:completed', {
      orderId: order.id,
      total: order.total,
    })

    // Emit stock:updated for each item
    for (const item of dto.items) {
      const sp = await this.prisma.storeProduct.findUnique({
        where: { storeId_productId: { storeId, productId: item.productId } },
      })
      if (sp) {
        this.eventsGateway.emitToStore(storeId, 'stock:updated', {
          productId: item.productId,
          storeId,
          newQty: sp.stockQty,
        })
      }
    }

    return order
  }

  async holdOrder(orderId: string, storeId: string) {
    const order = await this.prisma.order.findFirst({
      where: { id: orderId, storeId, status: 'PENDING' },
    })
    if (!order) throw new NotFoundException('Pending order not found')
    return this.prisma.order.update({
      where: { id: orderId },
      data: { status: 'HELD' },
      include: { items: true },
    })
  }

  async resumeOrder(orderId: string, storeId: string) {
    const order = await this.prisma.order.findFirst({
      where: { id: orderId, storeId, status: 'HELD' },
      include: { items: true },
    })
    if (!order) throw new NotFoundException('Held order not found')
    await this.prisma.order.update({
      where: { id: orderId },
      data: { status: 'PENDING' },
    })
    return order
  }

  async getOrders(
    storeId: string,
    query: { page?: number; limit?: number; status?: string },
  ) {
    const page = query.page ?? 1
    const limit = query.limit ?? 20
    const skip = (page - 1) * limit

    const where: any = { storeId }
    if (query.status) where.status = query.status

    const [orders, total] = await Promise.all([
      this.prisma.order.findMany({
        where,
        include: { items: true, cashier: { select: { id: true, name: true } } },
        orderBy: { createdAt: 'desc' },
        skip,
        take: limit,
      }),
      this.prisma.order.aggregate({ where, _count: { id: true } }),
    ])

    return { orders, total: total._count.id, page, limit }
  }
}
```

- [ ] **Step 4: Create orders.controller.ts**

```typescript
// apps/api/src/orders/orders.controller.ts
import { Controller, Post, Get, Patch, Param, Body, Request, Query, UseGuards } from '@nestjs/common'
import { OrdersService } from './orders.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('orders')
@UseGuards(JwtAuthGuard, RolesGuard)
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  @Post()
  @Roles('CASHIER')
  createOrder(@Body() body: any, @Request() req: any) {
    return this.ordersService.createOrder(
      req.user.storeId,
      body.shiftId,
      req.user.sub,
      body,
    )
  }

  @Get()
  @Roles('CASHIER')
  getOrders(@Request() req: any, @Query() query: any) {
    return this.ordersService.getOrders(req.user.storeId, query)
  }

  @Patch(':id/hold')
  @Roles('CASHIER')
  holdOrder(@Param('id') id: string, @Request() req: any) {
    return this.ordersService.holdOrder(id, req.user.storeId)
  }

  @Patch(':id/resume')
  @Roles('CASHIER')
  resumeOrder(@Param('id') id: string, @Request() req: any) {
    return this.ordersService.resumeOrder(id, req.user.storeId)
  }
}
```

- [ ] **Step 5: Create orders.module.ts**

```typescript
// apps/api/src/orders/orders.module.ts
import { Module } from '@nestjs/common'
import { OrdersController } from './orders.controller'
import { OrdersService } from './orders.service'
import { PrismaModule } from '../prisma/prisma.module'
import { EventsModule } from '../events/events.module'

@Module({
  imports: [PrismaModule, EventsModule],
  controllers: [OrdersController],
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

- [ ] **Step 6: Register OrdersModule in AppModule**

- [ ] **Step 7: Run tests**

```bash
cd apps/api && npx jest orders.service.spec --no-coverage
```

Expected: PASS (1 suite, 2 tests).

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/orders/
git commit -m "feat: add orders module with atomic stock deduction and WebSocket events"
```

---

## Task 6: Cart store and product cache (Frontend)

- [ ] **Step 1: Create Zustand cart store**

```typescript
// apps/web/src/lib/cart-store.ts
import { create } from 'zustand'

export interface CartItem {
  productId: string
  productName: string
  unitPrice: number
  qty: number
  discount: number
}

interface CartState {
  items: CartItem[]
  discountAmount: number
  taxAmount: number
  note: string
  addItem: (item: Omit<CartItem, 'qty'>) => void
  removeItem: (productId: string) => void
  updateQty: (productId: string, qty: number) => void
  setDiscount: (amount: number) => void
  setTax: (amount: number) => void
  setNote: (note: string) => void
  clearCart: () => void
  subtotal: () => number
  total: () => number
}

export const useCartStore = create<CartState>((set, get) => ({
  items: [],
  discountAmount: 0,
  taxAmount: 0,
  note: '',

  addItem: (item) =>
    set((s) => {
      const existing = s.items.find((i) => i.productId === item.productId)
      if (existing) {
        return {
          items: s.items.map((i) =>
            i.productId === item.productId ? { ...i, qty: i.qty + 1 } : i,
          ),
        }
      }
      return { items: [...s.items, { ...item, qty: 1 }] }
    }),

  removeItem: (productId) =>
    set((s) => ({ items: s.items.filter((i) => i.productId !== productId) })),

  updateQty: (productId, qty) =>
    set((s) => ({
      items:
        qty <= 0
          ? s.items.filter((i) => i.productId !== productId)
          : s.items.map((i) => (i.productId === productId ? { ...i, qty } : i)),
    })),

  setDiscount: (discountAmount) => set({ discountAmount }),
  setTax: (taxAmount) => set({ taxAmount }),
  setNote: (note) => set({ note }),
  clearCart: () => set({ items: [], discountAmount: 0, taxAmount: 0, note: '' }),

  subtotal: () =>
    get().items.reduce((sum, i) => sum + (i.unitPrice - i.discount) * i.qty, 0),

  total: () => {
    const s = get()
    return s.subtotal() - s.discountAmount + s.taxAmount
  },
}))
```

- [ ] **Step 2: Create IndexedDB product cache**

Install idb if not already: `pnpm add idb --filter=web`

```typescript
// apps/web/src/lib/product-cache.ts
import { openDB, DBSchema } from 'idb'

interface PosDB extends DBSchema {
  products: {
    key: string
    value: {
      id: string
      name: string
      sku: string
      sellPrice: number
      stockQty: number
      categoryId: string | null
      imageUrls: string[]
      mainImageIndex: number
    }
    indexes: { 'by-name': string; 'by-sku': string }
  }
}

const DB_NAME = 'merxylab-pos'
const DB_VERSION = 1

async function getDb() {
  return openDB<PosDB>(DB_NAME, DB_VERSION, {
    upgrade(db) {
      const store = db.createObjectStore('products', { keyPath: 'id' })
      store.createIndex('by-name', 'name')
      store.createIndex('by-sku', 'sku')
    },
  })
}

export async function cacheProducts(products: PosDB['products']['value'][]) {
  const db = await getDb()
  const tx = db.transaction('products', 'readwrite')
  await Promise.all([...products.map((p) => tx.store.put(p)), tx.done])
}

export async function searchProductsOffline(query: string) {
  const db = await getDb()
  const all = await db.getAll('products')
  const q = query.toLowerCase()
  return all.filter(
    (p) => p.name.toLowerCase().includes(q) || p.sku.toLowerCase().includes(q),
  )
}

export async function updateCachedStock(productId: string, newQty: number) {
  const db = await getDb()
  const product = await db.get('products', productId)
  if (product) await db.put('products', { ...product, stockQty: newQty })
}
```

- [ ] **Step 3: Create Socket.IO client singleton**

Install socket.io-client if not already: `pnpm add socket.io-client --filter=web`

```typescript
// apps/web/src/lib/socket.ts
import { io, Socket } from 'socket.io-client'

let socket: Socket | null = null

export function getSocket(token: string): Socket {
  if (socket?.connected) return socket
  socket = io(process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001', {
    auth: { token },
    autoConnect: true,
    reconnection: true,
    reconnectionDelay: 2000,
  })
  return socket
}

export function disconnectSocket() {
  socket?.disconnect()
  socket = null
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/lib/
git commit -m "feat: add cart store (Zustand), IndexedDB product cache, Socket.IO client"
```

---

## Task 7: POS main screen (Frontend)

- [ ] **Step 1: Create OfflineBanner.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/OfflineBanner.tsx
'use client'
export function OfflineBanner({ isOffline }: { isOffline: boolean }) {
  if (!isOffline) return null
  return (
    <div className="w-full bg-yellow-500 text-white text-sm font-semibold text-center py-2 px-4">
      Connection lost — payment unavailable. Cart is still usable.
    </div>
  )
}
```

- [ ] **Step 2: Create CategoryChips.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/CategoryChips.tsx
'use client'
interface Category { id: string; name: string }

interface Props {
  categories: Category[]
  selected: string | null
  onSelect: (id: string | null) => void
}

export function CategoryChips({ categories, selected, onSelect }: Props) {
  return (
    <div className="flex gap-2 overflow-x-auto pb-2 no-scrollbar">
      <button
        onClick={() => onSelect(null)}
        className={`flex-shrink-0 px-4 py-2 rounded-full text-sm font-semibold min-h-[44px] transition-colors ${
          selected === null
            ? 'bg-primary text-white'
            : 'bg-surface border border-border text-text-muted hover:bg-canvas'
        }`}
      >
        All
      </button>
      {categories.map((cat) => (
        <button
          key={cat.id}
          onClick={() => onSelect(cat.id)}
          className={`flex-shrink-0 px-4 py-2 rounded-full text-sm font-semibold min-h-[44px] transition-colors ${
            selected === cat.id
              ? 'bg-primary text-white'
              : 'bg-surface border border-border text-text-muted hover:bg-canvas'
          }`}
        >
          {cat.name}
        </button>
      ))}
    </div>
  )
}
```

- [ ] **Step 3: Create ProductCard.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/ProductCard.tsx
'use client'
interface Props {
  id: string
  name: string
  sellPrice: number
  stockQty: number
  imageUrl?: string
  onAdd: () => void
}

export function ProductCard({ name, sellPrice, stockQty, imageUrl, onAdd }: Props) {
  const outOfStock = stockQty <= 0
  return (
    <button
      onClick={onAdd}
      disabled={outOfStock}
      className={`flex flex-col items-center gap-2 p-3 bg-surface border border-border rounded-xl text-left w-full transition-all min-h-[100px] ${
        outOfStock ? 'opacity-50 cursor-not-allowed' : 'hover:border-primary active:scale-95'
      }`}
    >
      <div className="w-full aspect-square rounded-lg bg-canvas overflow-hidden">
        {imageUrl ? (
          <img src={imageUrl} alt={name} className="w-full h-full object-cover" />
        ) : (
          <div className="w-full h-full flex items-center justify-content-center bg-canvas" />
        )}
      </div>
      <div className="w-full">
        <p className="text-xs font-semibold text-text-dark line-clamp-2 leading-tight">{name}</p>
        <p className="text-sm font-bold text-primary mt-1">{sellPrice.toLocaleString()} Ks</p>
        {outOfStock && <p className="text-[10px] text-danger font-semibold">Out of stock</p>}
      </div>
    </button>
  )
}
```

- [ ] **Step 4: Create ProductGrid.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/ProductGrid.tsx
'use client'
import { ProductCard } from './ProductCard'
import { useCartStore } from '@/lib/cart-store'

interface Product {
  id: string
  name: string
  sellPrice: number
  stockQty: number
  imageUrls: string[]
  mainImageIndex: number
  categoryId: string | null
}

interface Props {
  products: Product[]
  isLoading: boolean
}

export function ProductGrid({ products, isLoading }: Props) {
  const addItem = useCartStore((s) => s.addItem)

  if (isLoading) {
    return (
      <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-3">
        {Array.from({ length: 8 }).map((_, i) => (
          <div key={i} className="h-32 bg-surface rounded-xl animate-pulse border border-border" />
        ))}
      </div>
    )
  }

  if (products.length === 0) {
    return (
      <div className="flex flex-col items-center justify-center h-40 text-text-muted">
        <p className="text-sm">No products found</p>
      </div>
    )
  }

  return (
    <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-3">
      {products.map((p) => (
        <ProductCard
          key={p.id}
          id={p.id}
          name={p.name}
          sellPrice={p.sellPrice}
          stockQty={p.stockQty}
          imageUrl={p.imageUrls[p.mainImageIndex] ?? undefined}
          onAdd={() =>
            addItem({
              productId: p.id,
              productName: p.name,
              unitPrice: p.sellPrice,
              discount: 0,
            })
          }
        />
      ))}
    </div>
  )
}
```

- [ ] **Step 5: Create CartItem.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/CartItem.tsx
'use client'
import { useCartStore, CartItem as ICartItem } from '@/lib/cart-store'
import { Minus, Plus, Trash2 } from 'lucide-react'

interface Props { item: ICartItem }

export function CartItemRow({ item }: Props) {
  const { updateQty, removeItem } = useCartStore()
  const lineTotal = (item.unitPrice - item.discount) * item.qty

  return (
    <div className="flex items-center gap-2 py-2 border-b border-border last:border-0">
      <div className="flex-1 min-w-0">
        <p className="text-sm font-semibold text-text-dark truncate">{item.productName}</p>
        <p className="text-xs text-text-muted">{item.unitPrice.toLocaleString()} Ks</p>
      </div>
      <div className="flex items-center gap-1">
        <button
          onClick={() => updateQty(item.productId, item.qty - 1)}
          className="w-[44px] h-[44px] flex items-center justify-center rounded-lg bg-canvas border border-border text-text-muted hover:bg-canvas active:scale-95"
        >
          <Minus size={14} />
        </button>
        <span className="w-8 text-center text-sm font-bold">{item.qty}</span>
        <button
          onClick={() => updateQty(item.productId, item.qty + 1)}
          className="w-[44px] h-[44px] flex items-center justify-center rounded-lg bg-canvas border border-border text-text-muted hover:bg-canvas active:scale-95"
        >
          <Plus size={14} />
        </button>
      </div>
      <div className="text-right min-w-[72px]">
        <p className="text-sm font-bold text-text-dark">{lineTotal.toLocaleString()} Ks</p>
        <button
          onClick={() => removeItem(item.productId)}
          className="text-danger p-1 min-h-[44px] min-w-[44px] flex items-center justify-center"
        >
          <Trash2 size={14} />
        </button>
      </div>
    </div>
  )
}
```

- [ ] **Step 6: Create CartPanel.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/CartPanel.tsx
'use client'
import { useCartStore } from '@/lib/cart-store'
import { CartItemRow } from './CartItem'

interface Props {
  isOffline: boolean
  onPay: () => void
  onHold: () => void
}

export function CartPanel({ isOffline, onPay, onHold }: Props) {
  const { items, subtotal, total, discountAmount, taxAmount, clearCart } = useCartStore()
  const hasItems = items.length > 0

  return (
    <div className="flex flex-col h-full bg-surface border-l border-border">
      <div className="flex items-center justify-between p-4 border-b border-border">
        <h2 className="text-sm font-bold text-text-dark">Cart ({items.length})</h2>
        {hasItems && (
          <button onClick={clearCart} className="text-xs text-danger font-semibold min-h-[44px] px-2">
            Clear
          </button>
        )}
      </div>

      <div className="flex-1 overflow-y-auto px-4">
        {items.length === 0 ? (
          <div className="flex items-center justify-center h-full text-text-muted">
            <p className="text-sm">Cart is empty</p>
          </div>
        ) : (
          items.map((item) => <CartItemRow key={item.productId} item={item} />)
        )}
      </div>

      <div className="p-4 border-t border-border space-y-2">
        <div className="flex justify-between text-sm text-text-muted">
          <span>Subtotal</span>
          <span>{subtotal().toLocaleString()} Ks</span>
        </div>
        {discountAmount > 0 && (
          <div className="flex justify-between text-sm text-danger">
            <span>Discount</span>
            <span>−{discountAmount.toLocaleString()} Ks</span>
          </div>
        )}
        {taxAmount > 0 && (
          <div className="flex justify-between text-sm text-text-muted">
            <span>Tax</span>
            <span>{taxAmount.toLocaleString()} Ks</span>
          </div>
        )}
        <div className="flex justify-between text-base font-bold text-text-dark pt-1 border-t border-border">
          <span>Total</span>
          <span>{total().toLocaleString()} Ks</span>
        </div>

        <button
          onClick={onHold}
          disabled={!hasItems}
          className="w-full py-3 rounded-xl border border-border text-sm font-semibold text-text-dark bg-canvas hover:bg-border disabled:opacity-40 min-h-[44px]"
        >
          Hold (F4)
        </button>
        <button
          onClick={onPay}
          disabled={!hasItems || isOffline}
          className="w-full py-3 rounded-xl bg-primary text-white text-sm font-bold hover:bg-primary/90 disabled:opacity-40 min-h-[44px]"
        >
          {isOffline ? 'Offline — Payment Unavailable' : 'Pay (F5)'}
        </button>
      </div>
    </div>
  )
}
```

- [ ] **Step 7: Create POS main page**

```tsx
// apps/web/src/app/(pos)/pos/page.tsx
'use client'
import { useState, useEffect, useCallback } from 'react'
import { useQuery } from '@tanstack/react-query'
import { CategoryChips } from './_components/CategoryChips'
import { ProductGrid } from './_components/ProductGrid'
import { CartPanel } from './_components/CartPanel'
import { OfflineBanner } from './_components/OfflineBanner'
import { PaymentModal } from './_components/PaymentModal'
import { cacheProducts, searchProductsOffline } from '@/lib/product-cache'
import { getSocket } from '@/lib/socket'
import { useCartStore } from '@/lib/cart-store'
import { apiFetch } from '@/lib/api'
import { useAuthStore } from '@/lib/auth-store'
import { Search } from 'lucide-react'

export default function POSPage() {
  const [isOffline, setIsOffline] = useState(false)
  const [selectedCategory, setSelectedCategory] = useState<string | null>(null)
  const [search, setSearch] = useState('')
  const [showPayment, setShowPayment] = useState(false)
  const { token } = useAuthStore()

  // Online/offline detection
  useEffect(() => {
    const handleOffline = () => setIsOffline(true)
    const handleOnline = () => setIsOffline(false)
    window.addEventListener('offline', handleOffline)
    window.addEventListener('online', handleOnline)
    return () => {
      window.removeEventListener('offline', handleOffline)
      window.removeEventListener('online', handleOnline)
    }
  }, [])

  // WebSocket
  useEffect(() => {
    if (!token) return
    const socket = getSocket(token)
    socket.on('stock:updated', ({ productId, newQty }: any) => {
      // Update cached product stock
    })
    return () => { socket.off('stock:updated') }
  }, [token])

  const { data: products, isLoading } = useQuery({
    queryKey: ['pos-products', selectedCategory, search],
    queryFn: async () => {
      if (isOffline) return searchProductsOffline(search)
      const params = new URLSearchParams()
      if (selectedCategory) params.set('categoryId', selectedCategory)
      if (search) params.set('search', search)
      const res = await apiFetch(`/products?${params.toString()}`)
      const data = res.data
      await cacheProducts(data)
      return data
    },
    staleTime: 30_000,
  })

  const { data: categories } = useQuery({
    queryKey: ['categories'],
    queryFn: () => apiFetch('/categories').then((r) => r.data),
    staleTime: 60_000,
  })

  // Keyboard shortcuts
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'F5') { e.preventDefault(); setShowPayment(true) }
      if (e.key === 'F2') { e.preventDefault(); document.getElementById('pos-search')?.focus() }
    }
    window.addEventListener('keydown', handler)
    return () => window.removeEventListener('keydown', handler)
  }, [])

  return (
    <div className="flex flex-col h-screen bg-canvas overflow-hidden">
      <OfflineBanner isOffline={isOffline} />

      <div className="flex flex-1 overflow-hidden">
        {/* Left: product area */}
        <div className="flex-1 flex flex-col overflow-hidden p-4 gap-3">
          {/* Search */}
          <div className="relative">
            <Search size={16} className="absolute left-3 top-1/2 -translate-y-1/2 text-text-muted" />
            <input
              id="pos-search"
              type="text"
              value={search}
              onChange={(e) => setSearch(e.target.value)}
              placeholder="Search products or scan barcode… (F2)"
              className="w-full pl-9 pr-4 py-3 bg-surface border border-border rounded-xl text-sm focus:outline-none focus:border-primary min-h-[44px]"
              style={{ fontSize: '16px' }}
            />
          </div>

          {/* Category chips */}
          <CategoryChips
            categories={categories ?? []}
            selected={selectedCategory}
            onSelect={setSelectedCategory}
          />

          {/* Product grid */}
          <div className="flex-1 overflow-y-auto">
            <ProductGrid products={products ?? []} isLoading={isLoading} />
          </div>
        </div>

        {/* Right: cart */}
        <div className="w-80 lg:w-96 flex-shrink-0">
          <CartPanel
            isOffline={isOffline}
            onPay={() => setShowPayment(true)}
            onHold={() => { /* handle hold */ }}
          />
        </div>
      </div>

      {showPayment && (
        <PaymentModal onClose={() => setShowPayment(false)} />
      )}
    </div>
  )
}
```

- [ ] **Step 8: Commit**

```bash
git add apps/web/src/app/(pos)/pos/ apps/web/src/lib/
git commit -m "feat: add POS main screen with product grid, cart, offline detection"
```

---

## Task 8: Payment modal and receipt preview

- [ ] **Step 1: Create PaymentModal.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/PaymentModal.tsx
'use client'
import { useState } from 'react'
import { useCartStore } from '@/lib/cart-store'
import { apiFetch } from '@/lib/api'
import { useAuthStore } from '@/lib/auth-store'
import { ReceiptPreview } from './ReceiptPreview'

interface Props { onClose: () => void }

export function PaymentModal({ onClose }: Props) {
  const [method, setMethod] = useState<'CASH' | 'CARD'>('CASH')
  const [amountTendered, setAmountTendered] = useState('')
  const [isLoading, setIsLoading] = useState(false)
  const [completedOrder, setCompletedOrder] = useState<any>(null)
  const { items, total, discountAmount, taxAmount, clearCart } = useCartStore()
  const { activeShiftId } = useAuthStore()

  const totalAmt = total()
  const change = method === 'CASH' ? (parseInt(amountTendered || '0') - totalAmt) : 0

  async function handlePay() {
    setIsLoading(true)
    try {
      const res = await apiFetch('/orders', {
        method: 'POST',
        body: JSON.stringify({
          shiftId: activeShiftId,
          items: items.map((i) => ({
            productId: i.productId,
            productName: i.productName,
            qty: i.qty,
            unitPrice: i.unitPrice,
            discount: i.discount,
          })),
          paymentMethod: method,
          amountTendered: method === 'CASH' ? parseInt(amountTendered) : undefined,
          discountAmount,
          taxAmount,
        }),
      })
      setCompletedOrder(res.data)
    } finally {
      setIsLoading(false)
    }
  }

  function handleDone() {
    clearCart()
    onClose()
  }

  if (completedOrder) {
    return <ReceiptPreview order={completedOrder} onDone={handleDone} />
  }

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
      <div className="bg-surface rounded-2xl w-full max-w-sm shadow-2xl overflow-hidden">
        <div className="p-5 border-b border-border">
          <h2 className="text-base font-bold text-text-dark">Payment</h2>
        </div>

        <div className="p-5 space-y-4">
          {/* Method toggle */}
          <div className="flex gap-2">
            {(['CASH', 'CARD'] as const).map((m) => (
              <button
                key={m}
                onClick={() => setMethod(m)}
                className={`flex-1 py-3 rounded-xl font-semibold text-sm min-h-[44px] transition-colors ${
                  method === m
                    ? 'bg-primary text-white'
                    : 'bg-canvas border border-border text-text-dark'
                }`}
              >
                {m}
              </button>
            ))}
          </div>

          {/* Total */}
          <div className="bg-canvas rounded-xl p-4 text-center">
            <p className="text-xs text-text-muted font-semibold uppercase tracking-wide mb-1">Total</p>
            <p className="text-3xl font-bold text-text-dark">{totalAmt.toLocaleString()} Ks</p>
          </div>

          {/* Cash tendered */}
          {method === 'CASH' && (
            <div className="space-y-2">
              <label className="text-xs font-bold uppercase tracking-wide text-text-muted">
                Amount Tendered
              </label>
              <input
                type="number"
                value={amountTendered}
                onChange={(e) => setAmountTendered(e.target.value)}
                placeholder="Enter amount"
                className="w-full px-4 py-3 bg-white border border-border rounded-xl text-base focus:outline-none focus:border-primary min-h-[44px]"
                style={{ fontSize: '16px' }}
                autoFocus
              />
              {amountTendered && change >= 0 && (
                <div className="flex justify-between text-sm font-semibold bg-primary/10 rounded-lg p-3">
                  <span className="text-text-muted">Change</span>
                  <span className="text-primary">{change.toLocaleString()} Ks</span>
                </div>
              )}
            </div>
          )}
        </div>

        <div className="p-5 border-t border-border flex gap-3">
          <button
            onClick={onClose}
            className="flex-1 py-3 rounded-xl border border-border text-sm font-semibold min-h-[44px]"
          >
            Cancel
          </button>
          <button
            onClick={handlePay}
            disabled={isLoading || (method === 'CASH' && change < 0)}
            className="flex-1 py-3 rounded-xl bg-primary text-white text-sm font-bold min-h-[44px] disabled:opacity-40"
          >
            {isLoading ? 'Processing…' : 'Complete Sale'}
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create ReceiptPreview.tsx**

```tsx
// apps/web/src/app/(pos)/pos/_components/ReceiptPreview.tsx
'use client'

interface OrderItem {
  productName: string
  qty: number
  unitPrice: number
  total: number
}

interface Order {
  orderNumber: number
  createdAt: string
  paymentMethod: string
  amountTendered: number | null
  changeAmount: number | null
  subtotal: number
  discountAmount: number
  taxAmount: number
  total: number
  items: OrderItem[]
}

interface Props { order: Order; onDone: () => void }

export function ReceiptPreview({ order, onDone }: Props) {
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
      <div className="bg-white rounded-2xl w-full max-w-xs shadow-2xl overflow-hidden">
        <div className="p-5 text-center border-b border-border">
          <p className="text-xs text-text-muted">MerxyLab POS</p>
          <p className="text-sm font-bold mt-1">Order #{String(order.orderNumber).padStart(4, '0')}</p>
          <p className="text-xs text-text-muted">
            {new Date(order.createdAt).toLocaleString()}
          </p>
        </div>

        <div className="p-4 space-y-2 max-h-60 overflow-y-auto">
          {order.items.map((item, i) => (
            <div key={i} className="flex justify-between text-sm">
              <span className="flex-1 truncate pr-2">{item.productName} × {item.qty}</span>
              <span className="font-semibold">{item.total.toLocaleString()} Ks</span>
            </div>
          ))}
        </div>

        <div className="p-4 border-t border-border space-y-1">
          <div className="flex justify-between text-sm text-text-muted">
            <span>Subtotal</span><span>{order.subtotal.toLocaleString()} Ks</span>
          </div>
          {order.discountAmount > 0 && (
            <div className="flex justify-between text-sm text-danger">
              <span>Discount</span><span>−{order.discountAmount.toLocaleString()} Ks</span>
            </div>
          )}
          <div className="flex justify-between text-base font-bold border-t border-border pt-2">
            <span>Total</span><span>{order.total.toLocaleString()} Ks</span>
          </div>
          {order.paymentMethod === 'CASH' && (
            <>
              <div className="flex justify-between text-sm text-text-muted">
                <span>Tendered</span><span>{order.amountTendered?.toLocaleString()} Ks</span>
              </div>
              <div className="flex justify-between text-sm font-semibold text-primary">
                <span>Change</span><span>{order.changeAmount?.toLocaleString()} Ks</span>
              </div>
            </>
          )}
        </div>

        <div className="p-4 border-t border-border flex gap-2">
          <button
            onClick={() => window.print()}
            className="flex-1 py-3 rounded-xl border border-border text-sm font-semibold min-h-[44px]"
          >
            Print
          </button>
          <button
            onClick={onDone}
            className="flex-1 py-3 rounded-xl bg-primary text-white text-sm font-bold min-h-[44px]"
          >
            Done
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(pos)/pos/_components/PaymentModal.tsx \
        apps/web/src/app/(pos)/pos/_components/ReceiptPreview.tsx
git commit -m "feat: add payment modal and receipt preview"
```

---

## Task 9: Open/close shift screens and held orders

- [ ] **Step 1: Create open shift page**

```tsx
// apps/web/src/app/(pos)/pos/shift/open/page.tsx
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { apiFetch } from '@/lib/api'
import { useAuthStore } from '@/lib/auth-store'

export default function OpenShiftPage() {
  const [openingCash, setOpeningCash] = useState('')
  const [notes, setNotes] = useState('')
  const [isLoading, setIsLoading] = useState(false)
  const router = useRouter()
  const { setActiveShiftId } = useAuthStore()

  async function handleOpen() {
    setIsLoading(true)
    try {
      const res = await apiFetch('/shifts/open', {
        method: 'POST',
        body: JSON.stringify({ openingCash: parseInt(openingCash || '0'), notes }),
      })
      setActiveShiftId(res.data.id)
      router.push('/pos')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <div className="min-h-screen bg-canvas flex items-center justify-center p-4">
      <div className="bg-surface border border-border rounded-2xl p-6 w-full max-w-sm shadow-lg">
        <h1 className="text-lg font-bold text-text-dark mb-1">Open Shift</h1>
        <p className="text-sm text-text-muted mb-6">Enter your opening cash count to start selling.</p>

        <div className="space-y-4">
          <div>
            <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">
              Opening Cash
            </label>
            <div className="flex">
              <input
                type="number"
                value={openingCash}
                onChange={(e) => setOpeningCash(e.target.value)}
                placeholder="0"
                className="flex-1 px-4 py-3 border border-border rounded-l-xl text-base focus:outline-none focus:border-primary min-h-[44px]"
                style={{ fontSize: '16px' }}
                autoFocus
              />
              <span className="px-4 bg-canvas border border-l-0 border-border rounded-r-xl flex items-center text-sm font-bold text-text-muted">
                Ks
              </span>
            </div>
          </div>
          <div>
            <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">
              Notes (optional)
            </label>
            <textarea
              value={notes}
              onChange={(e) => setNotes(e.target.value)}
              placeholder="Any notes for this shift…"
              className="w-full px-4 py-3 border border-border rounded-xl text-sm focus:outline-none focus:border-primary resize-none h-20"
            />
          </div>
          <button
            onClick={handleOpen}
            disabled={isLoading}
            className="w-full py-3 bg-primary text-white rounded-xl font-bold text-sm min-h-[44px] disabled:opacity-40"
          >
            {isLoading ? 'Opening…' : 'Open Shift & Start Selling'}
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create close shift page**

```tsx
// apps/web/src/app/(pos)/pos/shift/close/page.tsx
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { useQuery } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'
import { useAuthStore } from '@/lib/auth-store'

export default function CloseShiftPage() {
  const [closingCash, setClosingCash] = useState('')
  const [notes, setNotes] = useState('')
  const [isLoading, setIsLoading] = useState(false)
  const router = useRouter()
  const { activeShiftId, setActiveShiftId } = useAuthStore()

  const { data: summary } = useQuery({
    queryKey: ['shift-summary', activeShiftId],
    queryFn: () => apiFetch(`/shifts/${activeShiftId}/summary`).then((r) => r.data),
    enabled: !!activeShiftId,
  })

  async function handleClose() {
    if (!activeShiftId) return
    setIsLoading(true)
    try {
      await apiFetch(`/shifts/${activeShiftId}/close`, {
        method: 'PATCH',
        body: JSON.stringify({ closingCash: parseInt(closingCash || '0'), notes }),
      })
      setActiveShiftId(null)
      router.push('/pos/shift/open')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <div className="min-h-screen bg-canvas flex items-center justify-center p-4">
      <div className="bg-surface border border-border rounded-2xl p-6 w-full max-w-sm shadow-lg">
        <h1 className="text-lg font-bold text-text-dark mb-1">Close Shift</h1>
        {summary && (
          <div className="bg-canvas rounded-xl p-4 mb-5 space-y-2">
            <div className="flex justify-between text-sm">
              <span className="text-text-muted">Orders completed</span>
              <span className="font-bold">{summary.orderCount}</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-text-muted">Total sales</span>
              <span className="font-bold text-primary">{summary.totalSales?.toLocaleString()} Ks</span>
            </div>
            <div className="flex justify-between text-sm">
              <span className="text-text-muted">Opening cash</span>
              <span>{summary.openingCash?.toLocaleString()} Ks</span>
            </div>
          </div>
        )}
        <div className="space-y-4">
          <div>
            <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">
              Closing Cash Count
            </label>
            <div className="flex">
              <input
                type="number"
                value={closingCash}
                onChange={(e) => setClosingCash(e.target.value)}
                placeholder="0"
                className="flex-1 px-4 py-3 border border-border rounded-l-xl text-base focus:outline-none focus:border-primary min-h-[44px]"
                style={{ fontSize: '16px' }}
                autoFocus
              />
              <span className="px-4 bg-canvas border border-l-0 border-border rounded-r-xl flex items-center text-sm font-bold text-text-muted">Ks</span>
            </div>
          </div>
          <button
            onClick={handleClose}
            disabled={isLoading}
            className="w-full py-3 bg-danger text-white rounded-xl font-bold text-sm min-h-[44px] disabled:opacity-40"
          >
            {isLoading ? 'Closing…' : 'Close Shift'}
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create held orders page**

```tsx
// apps/web/src/app/(pos)/pos/held/page.tsx
'use client'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { useRouter } from 'next/navigation'
import { apiFetch } from '@/lib/api'
import { useCartStore } from '@/lib/cart-store'

export default function HeldOrdersPage() {
  const router = useRouter()
  const queryClient = useQueryClient()
  const { addItem, clearCart } = useCartStore()

  const { data } = useQuery({
    queryKey: ['held-orders'],
    queryFn: () => apiFetch('/orders?status=HELD').then((r) => r.data),
  })

  const resumeMutation = useMutation({
    mutationFn: (orderId: string) =>
      apiFetch(`/orders/${orderId}/resume`, { method: 'PATCH' }).then((r) => r.data),
    onSuccess: (order) => {
      clearCart()
      for (const item of order.items) {
        addItem({
          productId: item.productId,
          productName: item.productName,
          unitPrice: item.unitPrice,
          discount: item.discount,
        })
        // Set qty properly after adding
      }
      queryClient.invalidateQueries({ queryKey: ['held-orders'] })
      router.push('/pos')
    },
  })

  return (
    <div className="p-6 max-w-lg mx-auto">
      <h1 className="text-lg font-bold text-text-dark mb-4">Held Orders</h1>
      {!data?.orders?.length ? (
        <p className="text-sm text-text-muted">No held orders.</p>
      ) : (
        <div className="space-y-3">
          {data.orders.map((order: any) => (
            <div
              key={order.id}
              className="bg-surface border border-border rounded-xl p-4 flex items-center justify-between"
            >
              <div>
                <p className="text-sm font-bold">Order #{String(order.orderNumber).padStart(4, '0')}</p>
                <p className="text-xs text-text-muted">{order.items.length} items · {order.total.toLocaleString()} Ks</p>
              </div>
              <button
                onClick={() => resumeMutation.mutate(order.id)}
                className="px-4 py-2 bg-primary text-white rounded-lg text-sm font-semibold min-h-[44px]"
              >
                Resume
              </button>
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/app/(pos)/
git commit -m "feat: add shift open/close and held orders screens"
```

---

## Task 10: End-to-end smoke test

- [ ] **Step 1: Start dev stack**

```bash
pnpm dev
```

- [ ] **Step 2: Verify POS flow manually**

1. Log in as cashier → redirected to `/pos/shift/open`
2. Enter opening cash `5000` → click Open Shift
3. Confirm redirect to `/pos`
4. Search for a product → confirm it appears in the grid
5. Click a product → confirm it appears in the cart
6. Adjust qty up/down — verify cart total recalculates in Ks
7. Click Pay → Payment modal opens → select Cash → enter tendered amount
8. Click Complete Sale → receipt shows correct change in Ks
9. Click Done → cart clears → back to POS grid
10. Click Hold on a new cart → navigate to `/pos/held` → verify order appears → Resume

- [ ] **Step 3: Run all API tests**

```bash
cd apps/api && npx jest --no-coverage
```

Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "test: verify Phase 3 POS selling smoke test passes"
```
