# Phase 4 — Refunds & History Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the refund flow, order history screen, and cashier shift summary — allowing managers to issue partial/full refunds with automatic stock restoration.

**Architecture:** Refunds are store-scoped and require `manager+` role. Each `RefundItem` links back to an `OrderItem`. Refunding a trackable product creates a `StockMovement` of type `RECEIVE` and increments `StoreProduct.stockQty` atomically in the same Prisma transaction. Order history is paginated and searchable by cashier, date range, and status.

**Tech Stack:** NestJS refunds module, Prisma transactions, Next.js POS panel pages, TanStack Table, React Hook Form + Zod

**Depends on:** Phase 3 complete (orders, shifts, WebSocket gateway)

---

## File Map

```
apps/api/src/
├── prisma/schema.prisma                       # add Refund, RefundItem models
└── refunds/
    ├── refunds.module.ts
    ├── refunds.controller.ts
    ├── refunds.service.ts
    └── refunds.service.spec.ts

apps/web/src/app/(pos)/
├── pos/history/
│   └── page.tsx                               # order history list + detail modal
└── pos/refund/
    └── [orderId]/page.tsx                     # refund screen for a specific order
```

---

## Task 1: Extend Prisma schema — Refund, RefundItem

- [ ] **Step 1: Append to schema.prisma**

```prisma
model Refund {
  id         String       @id @default(cuid())
  orderId    String       @map("order_id")
  storeId    String       @map("store_id")
  cashierId  String       @map("cashier_id")
  reason     String
  total      Int
  createdAt  DateTime     @default(now()) @map("created_at")

  order      Order        @relation(fields: [orderId], references: [id])
  store      Store        @relation(fields: [storeId], references: [id])
  cashier    User         @relation(fields: [cashierId], references: [id])
  items      RefundItem[]

  @@map("refunds")
}

model RefundItem {
  id          String    @id @default(cuid())
  refundId    String    @map("refund_id")
  orderItemId String    @map("order_item_id")
  qty         Int
  amount      Int

  refund      Refund    @relation(fields: [refundId], references: [id])
  orderItem   OrderItem @relation(fields: [orderItemId], references: [id])

  @@map("refund_items")
}
```

- [ ] **Step 2: Run migration**

```bash
cd apps/api && npx prisma migrate dev --name add_refunds
```

Expected: Migration applied successfully.

- [ ] **Step 3: Commit**

```bash
git add apps/api/prisma/
git commit -m "feat: add Refund and RefundItem to Prisma schema"
```

---

## Task 2: Refunds service — TDD

- [ ] **Step 1: Write failing tests**

```typescript
// apps/api/src/refunds/refunds.service.spec.ts
import { Test } from '@nestjs/testing'
import { RefundsService } from './refunds.service'
import { PrismaService } from '../prisma/prisma.service'
import { BadRequestException, NotFoundException } from '@nestjs/common'

const mockPrisma = {
  $transaction: jest.fn(),
  order: { findFirst: jest.fn() },
  refund: { findMany: jest.fn() },
}

describe('RefundsService', () => {
  let service: RefundsService

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        RefundsService,
        { provide: PrismaService, useValue: mockPrisma },
      ],
    }).compile()
    service = module.get(RefundsService)
    jest.clearAllMocks()
  })

  describe('createRefund', () => {
    it('throws NotFoundException for unknown order', async () => {
      mockPrisma.order.findFirst.mockResolvedValue(null)
      await expect(
        service.createRefund('store1', 'cashier1', { orderId: 'o1', reason: 'r', items: [] }),
      ).rejects.toThrow(NotFoundException)
    })

    it('throws BadRequestException if refund qty exceeds order qty', async () => {
      mockPrisma.order.findFirst.mockResolvedValue({
        id: 'o1',
        status: 'COMPLETED',
        items: [{ id: 'oi1', productId: 'p1', qty: 2, total: 3000, product: { isTrackable: true } }],
      })
      mockPrisma.$transaction.mockImplementation(async (fn: any) => {
        const tx = {
          refund: { findMany: jest.fn().mockResolvedValue([]) },
          storeProduct: { findUnique: jest.fn(), update: jest.fn() },
          stockMovement: { create: jest.fn() },
          order: { update: jest.fn() },
          refund: { create: jest.fn() },
        }
        return fn(tx)
      })
      await expect(
        service.createRefund('store1', 'cashier1', {
          orderId: 'o1',
          reason: 'defect',
          items: [{ orderItemId: 'oi1', qty: 5, amount: 7500 }],
        }),
      ).rejects.toThrow(BadRequestException)
    })

    it('restores stock and creates refund atomically', async () => {
      const order = {
        id: 'o1',
        status: 'COMPLETED',
        items: [{ id: 'oi1', productId: 'p1', qty: 2, total: 3000, product: { isTrackable: true } }],
      }
      mockPrisma.order.findFirst.mockResolvedValue(order)
      const createdRefund = { id: 'r1', total: 1500 }
      mockPrisma.$transaction.mockImplementation(async (fn: any) => {
        const tx = {
          refund: {
            findMany: jest.fn().mockResolvedValue([]),
            create: jest.fn().mockResolvedValue(createdRefund),
          },
          storeProduct: {
            findUnique: jest.fn().mockResolvedValue({ stockQty: 5 }),
            update: jest.fn(),
          },
          stockMovement: { create: jest.fn() },
          order: { update: jest.fn() },
        }
        return fn(tx)
      })
      const result = await service.createRefund('store1', 'cashier1', {
        orderId: 'o1',
        reason: 'defect',
        items: [{ orderItemId: 'oi1', qty: 1, amount: 1500 }],
      })
      expect(result).toEqual(createdRefund)
    })
  })
})
```

- [ ] **Step 2: Run test to verify failure**

```bash
cd apps/api && npx jest refunds.service.spec --no-coverage
```

Expected: FAIL — `RefundsService` not found.

- [ ] **Step 3: Implement refunds.service.ts**

```typescript
// apps/api/src/refunds/refunds.service.ts
import { Injectable, BadRequestException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

interface RefundItemInput {
  orderItemId: string
  qty: number
  amount: number
}

interface CreateRefundDto {
  orderId: string
  reason: string
  items: RefundItemInput[]
}

@Injectable()
export class RefundsService {
  constructor(private readonly prisma: PrismaService) {}

  async createRefund(storeId: string, cashierId: string, dto: CreateRefundDto) {
    const order = await this.prisma.order.findFirst({
      where: { id: dto.orderId, storeId, status: { in: ['COMPLETED'] } },
      include: { items: { include: { product: true } } },
    })
    if (!order) throw new NotFoundException('Order not found or not eligible for refund')

    return this.prisma.$transaction(async (tx) => {
      // Check already-refunded quantities
      const existingRefunds = await tx.refund.findMany({
        where: { orderId: dto.orderId },
        include: { items: true },
      })

      const alreadyRefunded: Record<string, number> = {}
      for (const r of existingRefunds) {
        for (const ri of r.items) {
          alreadyRefunded[ri.orderItemId] = (alreadyRefunded[ri.orderItemId] ?? 0) + ri.qty
        }
      }

      // Validate quantities and restore stock
      for (const refundItem of dto.items) {
        const orderItem = order.items.find((i) => i.id === refundItem.orderItemId)
        if (!orderItem) throw new BadRequestException(`Order item ${refundItem.orderItemId} not found`)

        const previouslyRefunded = alreadyRefunded[refundItem.orderItemId] ?? 0
        const maxRefundable = orderItem.qty - previouslyRefunded
        if (refundItem.qty > maxRefundable) {
          throw new BadRequestException(
            `Cannot refund ${refundItem.qty} of "${orderItem.productName}" — only ${maxRefundable} eligible`,
          )
        }

        if (orderItem.product.isTrackable) {
          await tx.stockMovement.create({
            data: {
              storeId,
              productId: orderItem.productId,
              type: 'RECEIVE',
              qtyChange: refundItem.qty,
              createdById: cashierId,
            },
          })
          await tx.storeProduct.update({
            where: { storeId_productId: { storeId, productId: orderItem.productId } },
            data: { stockQty: { increment: refundItem.qty } },
          })
        }
      }

      const total = dto.items.reduce((sum, i) => sum + i.amount, 0)

      // Mark order refunded if all items refunded
      const totalOrderQty = order.items.reduce((s, i) => s + i.qty, 0)
      const totalRefundedQty =
        dto.items.reduce((s, i) => s + i.qty, 0) +
        Object.values(alreadyRefunded).reduce((s, v) => s + v, 0)

      if (totalRefundedQty >= totalOrderQty) {
        await tx.order.update({ where: { id: dto.orderId }, data: { status: 'REFUNDED' } })
      }

      return tx.refund.create({
        data: {
          orderId: dto.orderId,
          storeId,
          cashierId,
          reason: dto.reason,
          total,
          items: { create: dto.items.map((i) => ({ orderItemId: i.orderItemId, qty: i.qty, amount: i.amount })) },
        },
        include: { items: true },
      })
    })
  }

  async getRefunds(storeId: string, query: { page?: number; limit?: number }) {
    const page = query.page ?? 1
    const limit = query.limit ?? 20
    const skip = (page - 1) * limit
    const [refunds, count] = await Promise.all([
      this.prisma.refund.findMany({
        where: { storeId },
        include: { items: true, cashier: { select: { id: true, name: true } } },
        orderBy: { createdAt: 'desc' },
        skip,
        take: limit,
      }),
      this.prisma.refund.count({ where: { storeId } }),
    ])
    return { refunds, total: count, page, limit }
  }
}
```

- [ ] **Step 4: Create refunds.controller.ts**

```typescript
// apps/api/src/refunds/refunds.controller.ts
import { Controller, Post, Get, Body, Request, Query, UseGuards } from '@nestjs/common'
import { RefundsService } from './refunds.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('refunds')
@UseGuards(JwtAuthGuard, RolesGuard)
export class RefundsController {
  constructor(private readonly refundsService: RefundsService) {}

  @Post()
  @Roles('MANAGER')
  createRefund(@Body() body: any, @Request() req: any) {
    return this.refundsService.createRefund(req.user.storeId, req.user.sub, body)
  }

  @Get()
  @Roles('MANAGER')
  getRefunds(@Request() req: any, @Query() query: any) {
    return this.refundsService.getRefunds(req.user.storeId, query)
  }
}
```

- [ ] **Step 5: Create refunds.module.ts and register in AppModule**

```typescript
// apps/api/src/refunds/refunds.module.ts
import { Module } from '@nestjs/common'
import { RefundsController } from './refunds.controller'
import { RefundsService } from './refunds.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [RefundsController],
  providers: [RefundsService],
  exports: [RefundsService],
})
export class RefundsModule {}
```

Add `RefundsModule` to `apps/api/src/app.module.ts` imports.

- [ ] **Step 6: Run tests**

```bash
cd apps/api && npx jest refunds.service.spec --no-coverage
```

Expected: PASS (3 tests).

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/refunds/
git commit -m "feat: add refunds module with atomic stock restoration"
```

---

## Task 3: Order history page (Frontend)

- [ ] **Step 1: Create order history page**

```tsx
// apps/web/src/app/(pos)/pos/history/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { useRouter } from 'next/navigation'
import { apiFetch } from '@/lib/api'
import { Search } from 'lucide-react'

const STATUS_LABELS: Record<string, string> = {
  COMPLETED: 'Completed',
  REFUNDED: 'Refunded',
  HELD: 'Held',
  CANCELLED: 'Cancelled',
}

const STATUS_COLORS: Record<string, string> = {
  COMPLETED: 'text-primary bg-primary/10',
  REFUNDED: 'text-danger bg-danger/10',
  HELD: 'text-warning bg-warning/10',
  CANCELLED: 'text-text-muted bg-canvas',
}

export default function OrderHistoryPage() {
  const [search, setSearch] = useState('')
  const [page, setPage] = useState(1)
  const router = useRouter()

  const { data, isLoading } = useQuery({
    queryKey: ['orders', page, search],
    queryFn: () =>
      apiFetch(`/orders?page=${page}&limit=20${search ? `&search=${search}` : ''}`).then(
        (r) => r.data,
      ),
    staleTime: 10_000,
  })

  return (
    <div className="p-4 max-w-2xl mx-auto">
      <div className="flex items-center justify-between mb-4">
        <h1 className="text-base font-bold text-text-dark">Order History</h1>
        <button
          onClick={() => router.back()}
          className="text-sm text-text-muted min-h-[44px] px-3"
        >
          ← Back
        </button>
      </div>

      <div className="relative mb-4">
        <Search size={14} className="absolute left-3 top-1/2 -translate-y-1/2 text-text-muted" />
        <input
          value={search}
          onChange={(e) => { setSearch(e.target.value); setPage(1) }}
          placeholder="Search by order number…"
          className="w-full pl-8 pr-4 py-2.5 bg-surface border border-border rounded-xl text-sm focus:outline-none focus:border-primary"
          style={{ fontSize: '16px' }}
        />
      </div>

      {isLoading ? (
        <div className="space-y-2">
          {Array.from({ length: 5 }).map((_, i) => (
            <div key={i} className="h-16 bg-surface rounded-xl animate-pulse border border-border" />
          ))}
        </div>
      ) : (
        <>
          <div className="space-y-2">
            {data?.orders?.map((order: any) => (
              <div
                key={order.id}
                className="bg-surface border border-border rounded-xl p-4 flex items-center justify-between cursor-pointer hover:border-primary transition-colors"
                onClick={() => router.push(`/pos/refund/${order.id}`)}
              >
                <div>
                  <p className="text-sm font-bold text-text-dark">
                    #{String(order.orderNumber).padStart(4, '0')}
                  </p>
                  <p className="text-xs text-text-muted">
                    {order.items.length} items · {new Date(order.createdAt).toLocaleString()}
                  </p>
                </div>
                <div className="text-right">
                  <p className="text-sm font-bold text-text-dark">
                    {order.total.toLocaleString()} Ks
                  </p>
                  <span
                    className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${STATUS_COLORS[order.status]}`}
                  >
                    {STATUS_LABELS[order.status]}
                  </span>
                </div>
              </div>
            ))}
          </div>

          {data?.total > 20 && (
            <div className="flex justify-center gap-4 mt-4">
              <button
                disabled={page === 1}
                onClick={() => setPage((p) => p - 1)}
                className="px-4 py-2 rounded-lg border border-border text-sm disabled:opacity-40 min-h-[44px]"
              >
                Previous
              </button>
              <span className="flex items-center text-sm text-text-muted">
                Page {page}
              </span>
              <button
                disabled={page * 20 >= data.total}
                onClick={() => setPage((p) => p + 1)}
                className="px-4 py-2 rounded-lg border border-border text-sm disabled:opacity-40 min-h-[44px]"
              >
                Next
              </button>
            </div>
          )}
        </>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/(pos)/pos/history/
git commit -m "feat: add order history page"
```

---

## Task 4: Refund screen (Frontend)

- [ ] **Step 1: Create refund page**

```tsx
// apps/web/src/app/(pos)/pos/refund/[orderId]/page.tsx
'use client'
import { useState } from 'react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { useParams, useRouter } from 'next/navigation'
import { apiFetch } from '@/lib/api'

export default function RefundPage() {
  const { orderId } = useParams<{ orderId: string }>()
  const router = useRouter()
  const queryClient = useQueryClient()
  const [reason, setReason] = useState('')
  const [selected, setSelected] = useState<Record<string, number>>({})

  const { data: order } = useQuery({
    queryKey: ['order', orderId],
    queryFn: () => apiFetch(`/orders/${orderId}`).then((r) => r.data),
  })

  const refundMutation = useMutation({
    mutationFn: (payload: any) =>
      apiFetch('/refunds', { method: 'POST', body: JSON.stringify(payload) }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['orders'] })
      router.push('/pos/history')
    },
  })

  function toggleItem(itemId: string, maxQty: number) {
    setSelected((prev) => {
      if (prev[itemId] != null) {
        const { [itemId]: _, ...rest } = prev
        return rest
      }
      return { ...prev, [itemId]: maxQty }
    })
  }

  function handleRefund() {
    const items = Object.entries(selected).map(([orderItemId, qty]) => {
      const item = order?.items.find((i: any) => i.id === orderItemId)
      return { orderItemId, qty, amount: Math.round((item?.total ?? 0) * (qty / item?.qty)) }
    })
    refundMutation.mutate({ orderId, reason, items })
  }

  const refundTotal = Object.entries(selected).reduce((sum, [itemId, qty]) => {
    const item = order?.items.find((i: any) => i.id === itemId)
    return sum + Math.round((item?.total ?? 0) * (qty / (item?.qty ?? 1)))
  }, 0)

  if (!order) return <div className="p-6 text-sm text-text-muted">Loading…</div>

  const isRefundable = order.status === 'COMPLETED'

  return (
    <div className="p-4 max-w-lg mx-auto">
      <div className="flex items-center justify-between mb-4">
        <h1 className="text-base font-bold text-text-dark">
          Refund — Order #{String(order.orderNumber).padStart(4, '0')}
        </h1>
        <button onClick={() => router.back()} className="text-sm text-text-muted min-h-[44px] px-3">
          ← Back
        </button>
      </div>

      {!isRefundable && (
        <div className="bg-warning/10 border border-warning rounded-xl p-3 text-sm text-warning font-semibold mb-4">
          This order cannot be refunded (status: {order.status})
        </div>
      )}

      <div className="space-y-2 mb-4">
        {order.items.map((item: any) => (
          <div
            key={item.id}
            onClick={() => isRefundable && toggleItem(item.id, item.qty)}
            className={`bg-surface border rounded-xl p-4 flex items-center justify-between transition-colors ${
              isRefundable ? 'cursor-pointer' : 'opacity-60'
            } ${selected[item.id] != null ? 'border-primary' : 'border-border'}`}
          >
            <div>
              <p className="text-sm font-semibold text-text-dark">{item.productName}</p>
              <p className="text-xs text-text-muted">Qty: {item.qty} · {item.unitPrice.toLocaleString()} Ks each</p>
            </div>
            <div className="text-right">
              <p className="text-sm font-bold">{item.total.toLocaleString()} Ks</p>
              {selected[item.id] != null && (
                <span className="text-[10px] font-bold text-primary">Selected</span>
              )}
            </div>
          </div>
        ))}
      </div>

      {isRefundable && Object.keys(selected).length > 0 && (
        <div className="space-y-3">
          <div>
            <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">
              Reason <span className="text-danger">*</span>
            </label>
            <input
              value={reason}
              onChange={(e) => setReason(e.target.value)}
              placeholder="e.g. Defective item, customer changed mind…"
              className="w-full px-4 py-3 border border-border rounded-xl text-sm focus:outline-none focus:border-primary min-h-[44px]"
              style={{ fontSize: '16px' }}
            />
          </div>

          <div className="bg-canvas rounded-xl p-3 flex justify-between text-sm font-bold">
            <span>Refund Total</span>
            <span className="text-danger">{refundTotal.toLocaleString()} Ks</span>
          </div>

          <button
            onClick={handleRefund}
            disabled={!reason.trim() || refundMutation.isPending}
            className="w-full py-3 bg-danger text-white rounded-xl font-bold text-sm min-h-[44px] disabled:opacity-40"
          >
            {refundMutation.isPending ? 'Processing…' : 'Issue Refund'}
          </button>
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Run all tests**

```bash
cd apps/api && npx jest --no-coverage
```

Expected: All tests pass.

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app/(pos)/pos/refund/
git commit -m "feat: add refund screen"
```
