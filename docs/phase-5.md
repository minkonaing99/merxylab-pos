# Phase 5 — Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the manager/owner dashboard — KPI aggregations, sales and product charts, staff activity, refund report, and CSV export.

**Architecture:** A `ReportsModule` exposes read-only `GET` endpoints that run aggregate queries against the `orders`, `order_items`, `refunds`, `shifts`, and `users` tables, all filtered by `storeId` from the JWT. Superadmin can pass an explicit `storeId` query param to view any store. Frontend uses Recharts for bar/line/donut charts and TanStack Table for data tables. Exports stream CSV via a server-side route.

**Tech Stack:** NestJS reports module, Prisma aggregate queries, Next.js dashboard panel, Recharts, TanStack Table, date-fns (date range helpers)

**Depends on:** Phase 3 complete (orders + shifts), Phase 4 complete (refunds)

---

## File Map

```
apps/api/src/
└── reports/
    ├── reports.module.ts
    ├── reports.controller.ts
    └── reports.service.ts

apps/web/src/
├── app/(dashboard)/
│   └── dashboard/
│       ├── page.tsx                           # Overview — KPI cards + live order feed
│       ├── sales/page.tsx                     # Sales analytics — bar/line charts
│       ├── products/page.tsx                  # Product performance table
│       ├── staff/page.tsx                     # Staff activity table
│       ├── refunds/page.tsx                   # Refund report table
│       └── _components/
│           ├── KpiCard.tsx
│           ├── SalesChart.tsx
│           ├── DateRangePicker.tsx
│           └── ExportButton.tsx
└── lib/reports-api.ts                         # typed fetch helpers for report endpoints
```

---

## Task 1: Reports service (NestJS)

- [ ] **Step 1: Create reports.service.ts**

```typescript
// apps/api/src/reports/reports.service.ts
import { Injectable } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class ReportsService {
  constructor(private readonly prisma: PrismaService) {}

  async getSalesSummary(
    storeId: string,
    from: Date,
    to: Date,
  ) {
    const [totals, byDay] = await Promise.all([
      this.prisma.order.aggregate({
        where: { storeId, status: 'COMPLETED', createdAt: { gte: from, lte: to } },
        _sum: { total: true, discountAmount: true, taxAmount: true },
        _count: { id: true },
      }),
      this.prisma.$queryRaw<{ date: string; revenue: number; count: number }[]>`
        SELECT
          DATE(created_at AT TIME ZONE 'UTC') AS date,
          SUM(total)::int                      AS revenue,
          COUNT(id)::int                       AS count
        FROM orders
        WHERE store_id = ${storeId}
          AND status = 'COMPLETED'
          AND created_at >= ${from}
          AND created_at <= ${to}
        GROUP BY DATE(created_at AT TIME ZONE 'UTC')
        ORDER BY date ASC
      `,
    ])

    return {
      totalRevenue: totals._sum.total ?? 0,
      totalOrders: totals._count.id,
      totalDiscount: totals._sum.discountAmount ?? 0,
      totalTax: totals._sum.taxAmount ?? 0,
      byDay,
    }
  }

  async getProductPerformance(storeId: string, from: Date, to: Date, limit = 20) {
    const rows = await this.prisma.$queryRaw<
      { productId: string; productName: string; qtySold: number; revenue: number }[]
    >`
      SELECT
        oi.product_id       AS "productId",
        oi.product_name     AS "productName",
        SUM(oi.qty)::int    AS "qtySold",
        SUM(oi.total)::int  AS revenue
      FROM order_items oi
      INNER JOIN orders o ON o.id = oi.order_id
      WHERE o.store_id = ${storeId}
        AND o.status = 'COMPLETED'
        AND o.created_at >= ${from}
        AND o.created_at <= ${to}
      GROUP BY oi.product_id, oi.product_name
      ORDER BY revenue DESC
      LIMIT ${limit}
    `
    return rows
  }

  async getStaffActivity(storeId: string, from: Date, to: Date) {
    const rows = await this.prisma.$queryRaw<
      { cashierId: string; name: string; orders: number; revenue: number; shifts: number }[]
    >`
      SELECT
        u.id                  AS "cashierId",
        u.name,
        COUNT(DISTINCT o.id)::int   AS orders,
        COALESCE(SUM(o.total),0)::int AS revenue,
        COUNT(DISTINCT s.id)::int   AS shifts
      FROM users u
      LEFT JOIN orders o
        ON o.cashier_id = u.id
        AND o.store_id = ${storeId}
        AND o.status = 'COMPLETED'
        AND o.created_at >= ${from}
        AND o.created_at <= ${to}
      LEFT JOIN shifts s
        ON s.user_id = u.id
        AND s.store_id = ${storeId}
        AND s.opened_at >= ${from}
        AND s.opened_at <= ${to}
      INNER JOIN user_store_roles usr
        ON usr.user_id = u.id AND usr.store_id = ${storeId}
      GROUP BY u.id, u.name
      ORDER BY revenue DESC
    `
    return rows
  }

  async getRefundSummary(storeId: string, from: Date, to: Date) {
    const [totals, items] = await Promise.all([
      this.prisma.refund.aggregate({
        where: { storeId, createdAt: { gte: from, lte: to } },
        _sum: { total: true },
        _count: { id: true },
      }),
      this.prisma.refund.findMany({
        where: { storeId, createdAt: { gte: from, lte: to } },
        include: {
          cashier: { select: { id: true, name: true } },
          items: true,
        },
        orderBy: { createdAt: 'desc' },
        take: 50,
      }),
    ])
    return {
      totalRefunds: totals._count.id,
      totalAmount: totals._sum.total ?? 0,
      items,
    }
  }

  async exportSalesCsv(storeId: string, from: Date, to: Date): Promise<string> {
    const orders = await this.prisma.order.findMany({
      where: { storeId, status: 'COMPLETED', createdAt: { gte: from, lte: to } },
      include: { cashier: { select: { name: true } }, items: true },
      orderBy: { createdAt: 'asc' },
    })

    const header = 'Order#,Date,Cashier,Items,Subtotal,Discount,Tax,Total,Payment\n'
    const rows = orders.map((o) =>
      [
        String(o.orderNumber).padStart(4, '0'),
        new Date(o.createdAt).toISOString(),
        o.cashier.name,
        o.items.length,
        o.subtotal,
        o.discountAmount,
        o.taxAmount,
        o.total,
        o.paymentMethod,
      ].join(','),
    )
    return header + rows.join('\n')
  }
}
```

- [ ] **Step 2: Create reports.controller.ts**

```typescript
// apps/api/src/reports/reports.controller.ts
import { Controller, Get, Query, Request, UseGuards, Res } from '@nestjs/common'
import { ReportsService } from './reports.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'
import { FastifyReply } from 'fastify'

function parseDateRange(from?: string, to?: string) {
  const now = new Date()
  const toDate = to ? new Date(to) : now
  const fromDate = from ? new Date(from) : new Date(now.getFullYear(), now.getMonth(), 1)
  return { from: fromDate, to: toDate }
}

@Controller('reports')
@UseGuards(JwtAuthGuard, RolesGuard)
export class ReportsController {
  constructor(private readonly reportsService: ReportsService) {}

  @Get('sales')
  @Roles('MANAGER')
  getSales(@Request() req: any, @Query() q: any) {
    const { from, to } = parseDateRange(q.from, q.to)
    const storeId = req.user.role === 'SUPERADMIN' && q.storeId ? q.storeId : req.user.storeId
    return this.reportsService.getSalesSummary(storeId, from, to)
  }

  @Get('products')
  @Roles('MANAGER')
  getProducts(@Request() req: any, @Query() q: any) {
    const { from, to } = parseDateRange(q.from, q.to)
    return this.reportsService.getProductPerformance(req.user.storeId, from, to)
  }

  @Get('staff')
  @Roles('MANAGER')
  getStaff(@Request() req: any, @Query() q: any) {
    const { from, to } = parseDateRange(q.from, q.to)
    return this.reportsService.getStaffActivity(req.user.storeId, from, to)
  }

  @Get('refunds')
  @Roles('MANAGER')
  getRefunds(@Request() req: any, @Query() q: any) {
    const { from, to } = parseDateRange(q.from, q.to)
    return this.reportsService.getRefundSummary(req.user.storeId, from, to)
  }

  @Get('sales/export')
  @Roles('MANAGER')
  async exportSales(
    @Request() req: any,
    @Query() q: any,
    @Res() res: FastifyReply,
  ) {
    const { from, to } = parseDateRange(q.from, q.to)
    const csv = await this.reportsService.exportSalesCsv(req.user.storeId, from, to)
    res
      .header('Content-Type', 'text/csv')
      .header('Content-Disposition', `attachment; filename="sales-${Date.now()}.csv"`)
      .send(csv)
  }
}
```

- [ ] **Step 3: Create reports.module.ts and register in AppModule**

```typescript
// apps/api/src/reports/reports.module.ts
import { Module } from '@nestjs/common'
import { ReportsController } from './reports.controller'
import { ReportsService } from './reports.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [ReportsController],
  providers: [ReportsService],
})
export class ReportsModule {}
```

Add `ReportsModule` to `apps/api/src/app.module.ts` imports.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/reports/
git commit -m "feat: add reports module (sales, products, staff, refunds, CSV export)"
```

---

## Task 2: Shared report fetch helpers (Frontend)

- [ ] **Step 1: Create reports-api.ts**

```typescript
// apps/web/src/lib/reports-api.ts
import { apiFetch } from './api'

export type DateRange = { from: string; to: string }

export function getSalesSummary(range: DateRange) {
  return apiFetch(`/reports/sales?from=${range.from}&to=${range.to}`).then((r) => r.data)
}

export function getProductPerformance(range: DateRange) {
  return apiFetch(`/reports/products?from=${range.from}&to=${range.to}`).then((r) => r.data)
}

export function getStaffActivity(range: DateRange) {
  return apiFetch(`/reports/staff?from=${range.from}&to=${range.to}`).then((r) => r.data)
}

export function getRefundSummary(range: DateRange) {
  return apiFetch(`/reports/refunds?from=${range.from}&to=${range.to}`).then((r) => r.data)
}

export function downloadSalesCsv(range: DateRange) {
  const url = `${process.env.NEXT_PUBLIC_API_URL}/reports/sales/export?from=${range.from}&to=${range.to}`
  const a = document.createElement('a')
  a.href = url
  a.download = `sales-${range.from}-${range.to}.csv`
  a.click()
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/lib/reports-api.ts
git commit -m "feat: add typed report fetch helpers"
```

---

## Task 3: Shared dashboard components

- [ ] **Step 1: Create KpiCard.tsx**

```tsx
// apps/web/src/app/(dashboard)/dashboard/_components/KpiCard.tsx
interface Props {
  label: string
  value: string
  delta?: number   // positive = up, negative = down
  sub?: string
}

export function KpiCard({ label, value, delta, sub }: Props) {
  return (
    <div className="bg-surface border border-border rounded-xl p-4">
      <p className="text-xs font-bold uppercase tracking-wide text-text-muted mb-1">{label}</p>
      <p className="text-2xl font-bold text-text-dark">{value}</p>
      {delta != null && (
        <p className={`text-xs font-semibold mt-1 ${delta >= 0 ? 'text-primary' : 'text-danger'}`}>
          {delta >= 0 ? '▲' : '▼'} {Math.abs(delta).toFixed(1)}% vs last period
        </p>
      )}
      {sub && <p className="text-xs text-text-muted mt-1">{sub}</p>}
    </div>
  )
}
```

- [ ] **Step 2: Create DateRangePicker.tsx**

```tsx
// apps/web/src/app/(dashboard)/dashboard/_components/DateRangePicker.tsx
'use client'
import { useState } from 'react'

export type DateRange = { from: string; to: string }

interface Props { value: DateRange; onChange: (r: DateRange) => void }

const PRESETS = [
  { label: 'Today', days: 0 },
  { label: '7 days', days: 7 },
  { label: '30 days', days: 30 },
  { label: '90 days', days: 90 },
]

function toIso(d: Date) { return d.toISOString().split('T')[0] }

export function DateRangePicker({ value, onChange }: Props) {
  return (
    <div className="flex flex-wrap items-center gap-2">
      {PRESETS.map((p) => {
        const to = new Date()
        const from = new Date()
        from.setDate(from.getDate() - p.days)
        const fromStr = toIso(from)
        const toStr = toIso(to)
        const active = value.from === fromStr && value.to === toStr
        return (
          <button
            key={p.label}
            onClick={() => onChange({ from: fromStr, to: toStr })}
            className={`px-3 py-1.5 rounded-lg text-xs font-semibold min-h-[36px] transition-colors ${
              active ? 'bg-primary text-white' : 'bg-surface border border-border text-text-muted hover:bg-canvas'
            }`}
          >
            {p.label}
          </button>
        )
      })}
      <input
        type="date"
        value={value.from}
        onChange={(e) => onChange({ ...value, from: e.target.value })}
        className="px-3 py-1.5 bg-surface border border-border rounded-lg text-xs focus:outline-none focus:border-primary min-h-[36px]"
      />
      <span className="text-text-muted text-xs">→</span>
      <input
        type="date"
        value={value.to}
        onChange={(e) => onChange({ ...value, to: e.target.value })}
        className="px-3 py-1.5 bg-surface border border-border rounded-lg text-xs focus:outline-none focus:border-primary min-h-[36px]"
      />
    </div>
  )
}
```

- [ ] **Step 3: Create SalesChart.tsx**

```tsx
// apps/web/src/app/(dashboard)/dashboard/_components/SalesChart.tsx
'use client'
import {
  ResponsiveContainer,
  BarChart,
  Bar,
  XAxis,
  YAxis,
  Tooltip,
  CartesianGrid,
} from 'recharts'

interface DataPoint { date: string; revenue: number; count: number }

interface Props { data: DataPoint[] }

export function SalesChart({ data }: Props) {
  return (
    <div className="bg-surface border border-border rounded-xl p-4">
      <p className="text-xs font-bold uppercase tracking-wide text-text-muted mb-4">Revenue by Day</p>
      <ResponsiveContainer width="100%" height={220}>
        <BarChart data={data} margin={{ top: 4, right: 4, left: 0, bottom: 0 }}>
          <CartesianGrid strokeDasharray="3 3" stroke="#d1d8e0" />
          <XAxis
            dataKey="date"
            tick={{ fontSize: 10, fill: '#718096' }}
            tickFormatter={(v) => v.slice(5)}
          />
          <YAxis
            tick={{ fontSize: 10, fill: '#718096' }}
            tickFormatter={(v) => `${(v / 1000).toFixed(0)}K`}
          />
          <Tooltip
            formatter={(value: number) => [`${value.toLocaleString()} Ks`, 'Revenue']}
            contentStyle={{ fontSize: 12, borderRadius: 8, border: '1px solid #d1d8e0' }}
          />
          <Bar dataKey="revenue" fill="#059669" radius={[4, 4, 0, 0]} />
        </BarChart>
      </ResponsiveContainer>
    </div>
  )
}
```

- [ ] **Step 4: Create ExportButton.tsx**

```tsx
// apps/web/src/app/(dashboard)/dashboard/_components/ExportButton.tsx
'use client'
import { Download } from 'lucide-react'
import { DateRange, downloadSalesCsv } from '@/lib/reports-api'

interface Props { range: DateRange }

export function ExportButton({ range }: Props) {
  return (
    <button
      onClick={() => downloadSalesCsv(range)}
      className="flex items-center gap-2 px-4 py-2 bg-surface border border-border rounded-xl text-sm font-semibold text-text-dark hover:bg-canvas min-h-[44px]"
    >
      <Download size={14} />
      Export CSV
    </button>
  )
}
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/app/(dashboard)/dashboard/_components/
git commit -m "feat: add dashboard KpiCard, SalesChart, DateRangePicker, ExportButton"
```

---

## Task 4: Dashboard overview page

- [ ] **Step 1: Create overview page**

```tsx
// apps/web/src/app/(dashboard)/dashboard/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { KpiCard } from './_components/KpiCard'
import { SalesChart } from './_components/SalesChart'
import { DateRangePicker, DateRange } from './_components/DateRangePicker'
import { ExportButton } from './_components/ExportButton'
import { getSalesSummary } from '@/lib/reports-api'

function defaultRange(): DateRange {
  const to = new Date()
  const from = new Date()
  from.setDate(from.getDate() - 30)
  return {
    from: from.toISOString().split('T')[0],
    to: to.toISOString().split('T')[0],
  }
}

export default function DashboardPage() {
  const [range, setRange] = useState<DateRange>(defaultRange)

  const { data, isLoading } = useQuery({
    queryKey: ['sales-summary', range],
    queryFn: () => getSalesSummary(range),
    staleTime: 30_000,
  })

  return (
    <div className="p-6 space-y-6">
      <div className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-lg font-bold text-text-dark">Overview</h1>
        <div className="flex items-center gap-3">
          <DateRangePicker value={range} onChange={setRange} />
          <ExportButton range={range} />
        </div>
      </div>

      {isLoading ? (
        <div className="grid grid-cols-2 lg:grid-cols-4 gap-4">
          {Array.from({ length: 4 }).map((_, i) => (
            <div key={i} className="h-24 bg-surface rounded-xl animate-pulse border border-border" />
          ))}
        </div>
      ) : (
        <div className="grid grid-cols-2 lg:grid-cols-4 gap-4">
          <KpiCard
            label="Total Revenue"
            value={`${(data?.totalRevenue ?? 0).toLocaleString()} Ks`}
          />
          <KpiCard
            label="Orders"
            value={String(data?.totalOrders ?? 0)}
          />
          <KpiCard
            label="Discount Given"
            value={`${(data?.totalDiscount ?? 0).toLocaleString()} Ks`}
          />
          <KpiCard
            label="Tax Collected"
            value={`${(data?.totalTax ?? 0).toLocaleString()} Ks`}
          />
        </div>
      )}

      {data?.byDay && <SalesChart data={data.byDay} />}
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/web/src/app/(dashboard)/dashboard/page.tsx
git commit -m "feat: add dashboard overview page with KPIs and sales chart"
```

---

## Task 5: Sales analytics, product performance, staff, refunds pages

- [ ] **Step 1: Create sales analytics page**

```tsx
// apps/web/src/app/(dashboard)/dashboard/sales/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { SalesChart } from '../_components/SalesChart'
import { DateRangePicker, DateRange } from '../_components/DateRangePicker'
import { ExportButton } from '../_components/ExportButton'
import { getSalesSummary } from '@/lib/reports-api'

function defaultRange(): DateRange {
  const to = new Date()
  const from = new Date()
  from.setDate(from.getDate() - 30)
  return { from: from.toISOString().split('T')[0], to: to.toISOString().split('T')[0] }
}

export default function SalesPage() {
  const [range, setRange] = useState<DateRange>(defaultRange)
  const { data } = useQuery({
    queryKey: ['sales-detail', range],
    queryFn: () => getSalesSummary(range),
    staleTime: 30_000,
  })

  return (
    <div className="p-6 space-y-6">
      <div className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-lg font-bold text-text-dark">Sales Analytics</h1>
        <div className="flex items-center gap-3">
          <DateRangePicker value={range} onChange={setRange} />
          <ExportButton range={range} />
        </div>
      </div>
      {data?.byDay && <SalesChart data={data.byDay} />}
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['Date', 'Orders', 'Revenue'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {data?.byDay?.map((row: any) => (
              <tr key={row.date} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 text-text-dark">{row.date}</td>
                <td className="px-4 py-3 text-text-muted">{row.count}</td>
                <td className="px-4 py-3 font-semibold">{row.revenue.toLocaleString()} Ks</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create product performance page**

```tsx
// apps/web/src/app/(dashboard)/dashboard/products/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { DateRangePicker, DateRange } from '../_components/DateRangePicker'
import { getProductPerformance } from '@/lib/reports-api'

function defaultRange(): DateRange {
  const to = new Date()
  const from = new Date()
  from.setDate(from.getDate() - 30)
  return { from: from.toISOString().split('T')[0], to: to.toISOString().split('T')[0] }
}

export default function ProductsReportPage() {
  const [range, setRange] = useState<DateRange>(defaultRange)
  const { data } = useQuery({
    queryKey: ['product-perf', range],
    queryFn: () => getProductPerformance(range),
    staleTime: 30_000,
  })

  return (
    <div className="p-6 space-y-6">
      <div className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-lg font-bold text-text-dark">Product Performance</h1>
        <DateRangePicker value={range} onChange={setRange} />
      </div>
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['#', 'Product', 'Qty Sold', 'Revenue'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(data ?? []).map((row: any, i: number) => (
              <tr key={row.productId} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 text-text-muted">{i + 1}</td>
                <td className="px-4 py-3 font-semibold text-text-dark">{row.productName}</td>
                <td className="px-4 py-3 text-text-muted">{row.qtySold}</td>
                <td className="px-4 py-3 font-bold text-primary">{row.revenue.toLocaleString()} Ks</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create staff activity page**

```tsx
// apps/web/src/app/(dashboard)/dashboard/staff/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { DateRangePicker, DateRange } from '../_components/DateRangePicker'
import { getStaffActivity } from '@/lib/reports-api'

function defaultRange(): DateRange {
  const to = new Date()
  const from = new Date()
  from.setDate(from.getDate() - 30)
  return { from: from.toISOString().split('T')[0], to: to.toISOString().split('T')[0] }
}

export default function StaffPage() {
  const [range, setRange] = useState<DateRange>(defaultRange)
  const { data } = useQuery({
    queryKey: ['staff-activity', range],
    queryFn: () => getStaffActivity(range),
    staleTime: 30_000,
  })

  return (
    <div className="p-6 space-y-6">
      <div className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-lg font-bold text-text-dark">Staff Activity</h1>
        <DateRangePicker value={range} onChange={setRange} />
      </div>
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['Cashier', 'Shifts', 'Orders', 'Revenue'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(data ?? []).map((row: any) => (
              <tr key={row.cashierId} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 font-semibold text-text-dark">{row.name}</td>
                <td className="px-4 py-3 text-text-muted">{row.shifts}</td>
                <td className="px-4 py-3 text-text-muted">{row.orders}</td>
                <td className="px-4 py-3 font-bold text-primary">{row.revenue.toLocaleString()} Ks</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create refunds report page**

```tsx
// apps/web/src/app/(dashboard)/dashboard/refunds/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { DateRangePicker, DateRange } from '../_components/DateRangePicker'
import { KpiCard } from '../_components/KpiCard'
import { getRefundSummary } from '@/lib/reports-api'

function defaultRange(): DateRange {
  const to = new Date()
  const from = new Date()
  from.setDate(from.getDate() - 30)
  return { from: from.toISOString().split('T')[0], to: to.toISOString().split('T')[0] }
}

export default function RefundsReportPage() {
  const [range, setRange] = useState<DateRange>(defaultRange)
  const { data } = useQuery({
    queryKey: ['refunds-report', range],
    queryFn: () => getRefundSummary(range),
    staleTime: 30_000,
  })

  return (
    <div className="p-6 space-y-6">
      <div className="flex flex-wrap items-center justify-between gap-4">
        <h1 className="text-lg font-bold text-text-dark">Refund Report</h1>
        <DateRangePicker value={range} onChange={setRange} />
      </div>
      <div className="grid grid-cols-2 gap-4">
        <KpiCard label="Total Refunds" value={String(data?.totalRefunds ?? 0)} />
        <KpiCard label="Refund Amount" value={`${(data?.totalAmount ?? 0).toLocaleString()} Ks`} />
      </div>
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['Date', 'Cashier', 'Reason', 'Amount'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(data?.items ?? []).map((r: any) => (
              <tr key={r.id} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 text-text-muted">{new Date(r.createdAt).toLocaleDateString()}</td>
                <td className="px-4 py-3">{r.cashier.name}</td>
                <td className="px-4 py-3 text-text-muted truncate max-w-[200px]">{r.reason}</td>
                <td className="px-4 py-3 font-bold text-danger">{r.total.toLocaleString()} Ks</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
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
git add apps/web/src/app/(dashboard)/
git commit -m "feat: add dashboard sales, products, staff, and refunds report pages"
```
