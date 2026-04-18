# MerxyLab POS — System Design Spec

**Date:** 2026-04-18  
**Status:** Approved  
**Repo:** https://github.com/minkonaing99/merxylab-pos.git  
**Currency:** Myanmar Kyat (Ks)

---

## 1. Project Overview

A multi-store Point-of-Sale web application with four panels: Cashier POS, Dashboard, Admin, and Stock. The system supports multiple stores sharing a global product catalog with per-store inventory levels, role-based access control, and real-time sync across terminals.

### Core Loop

```
Admin creates users/settings
→ Stock panel creates products and stock
→ Cashier sells products
→ Inventory auto-deducts
→ Dashboard shows reports
```

---

## 2. Tech Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 14 (App Router) |
| UI | Tailwind CSS + shadcn/ui |
| State | Zustand |
| Tables | TanStack Table v8 |
| Forms | React Hook Form + Zod |
| Charts | Recharts |
| Backend | NestJS + Fastify |
| ORM | Prisma |
| Database | PostgreSQL 16 |
| Cache / Sessions | Redis |
| Auth | JWT (access 15min + refresh 7d, Redis-backed) |
| Real-time | Socket.IO (NestJS gateway) |
| Job Queue | BullMQ (Redis-backed) |
| File Storage | S3-compatible (MinIO local, AWS S3 prod) |
| Monorepo | Turborepo |
| Package Manager | pnpm |

---

## 3. Repository Structure

```
merxylab-pos/
├── apps/
│   ├── web/                    # Next.js 14 (App Router)
│   │   └── src/app/
│   │       ├── (pos)/          # Cashier POS — cashier role+
│   │       ├── (dashboard)/    # Dashboard — manager role+
│   │       ├── (stock)/        # Stock panel — inventory role+
│   │       └── (admin)/        # Admin panel — admin role+
│   └── api/                    # NestJS + Fastify
│       └── src/
│           ├── auth/
│           ├── users/
│           ├── stores/
│           ├── products/
│           ├── categories/
│           ├── inventory/
│           ├── suppliers/
│           ├── purchase-orders/
│           ├── shifts/
│           ├── orders/
│           ├── refunds/
│           ├── reports/
│           ├── devices/
│           ├── audit-logs/
│           └── uploads/
├── packages/
│   ├── types/                  # Shared TypeScript interfaces & enums
│   └── utils/                  # Shared pure utilities (formatters, validators)
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

---

## 4. Design System

### Visual Style

- **Theme:** Clean light — cool gray canvas, dark sidebar, green primary CTA
- **Canvas:** `#edf0f4` (cool gray — not white)
- **Surface:** `#f7f9fb` (card/panel background)
- **Sidebar:** `#1a202c` (dark charcoal)

### Design Tokens

| Token | Value | Usage |
|---|---|---|
| `primary` | `#059669` | Pay button, active nav, positive deltas, active chip |
| `primary-light` | `#34d399` | Active nav text in sidebar |
| `primary-tint` | `#d1fae5` | Selected state backgrounds |
| `canvas` | `#edf0f4` | Page background |
| `surface` | `#f7f9fb` | Cards, panels, inputs |
| `sidebar-bg` | `#1a202c` | Sidebar background |
| `text-dark` | `#1a202c` | Primary text |
| `text-muted` | `#718096` | Labels, hints, secondary text |
| `border` | `#d1d8e0` | Borders throughout |
| `danger` | `#e53e3e` | Errors, delete actions |
| `warning` | `#d69e2e` | Low stock, caution |
| `info` | `#3182ce` | Info badges |

Green is used **only** for: primary action buttons, active nav items, positive KPI deltas, selected states. Everything else uses slate/charcoal.

### Touch & Responsive Rules

| Rule | Value |
|---|---|
| Minimum tap target | 44 × 44px |
| Form input font-size | 16px min (prevents iOS auto-zoom) |
| POS product card | min 100px wide, fluid grid |
| Cart qty buttons | 44px touch area |
| Sidebar on tablet | collapses to icon-only rail (64px wide) |
| Sidebar on mobile | hidden by default, opens as bottom sheet |
| POS on tablet landscape | full two-column layout unchanged |
| POS on tablet portrait | 3-column product grid, cart as bottom drawer |
| POS on mobile | graceful degradation (not primary target) |
| Modals | full-screen on mobile, centred sheet on tablet+ |
| Swipe gesture | swipe left on cart item → delete |

---

## 5. Multi-Store Architecture

- Every resource belongs to a `Store`
- Users can hold roles in multiple stores simultaneously
- `superadmin` sits above all stores for system-level config
- **Products are global** — shared catalog across all stores
- **Stock is per-store** — `StoreProduct` bridges product and store with `stock_qty`, `min_stock_qty`
- Reports always filter by `store_id` unless the user is `superadmin` viewing cross-store

---

## 6. Data Model

### Auth & Users

```
User
  id, name, email, password_hash, is_active, created_at, updated_at

Store  [global]
  id, name, address, phone, currency (default: Ks), timezone, is_active, created_at

UserStoreRole
  user_id → User
  store_id → Store
  role: enum(superadmin | admin | manager | cashier | inventory)
```

### Product Catalog (Global)

```
Product
  id, name, sku, barcode
  category_id → Category
  brand_id → Brand
  unit_id → Unit
  cost_price, sell_price (in Ks, integer)
  image_urls: string[]   -- up to 5, S3 keys
  main_image_index: int  -- which image is primary (0-based)
  is_active, is_trackable
  created_at, updated_at

Category
  id, name, slug, parent_id (self-ref, nested), sort_order

Brand
  id, name

Unit
  id, name, abbreviation   -- e.g. "Piece", "pcs"
```

### Inventory (Per-Store)

```
StoreProduct
  id, store_id → Store, product_id → Product
  stock_qty, min_stock_qty, max_stock_qty

StockMovement  [append-only ledger]
  id, store_id, product_id
  type: enum(sale | receive | adjust | transfer_out | transfer_in)
  qty_change (positive or negative)
  reference_id  -- order_id / adjustment_id / transfer_id
  note, created_by → User, created_at

StockAdjustment
  id, store_id, product_id
  old_qty, new_qty, reason
  created_by → User, created_at

StockTransfer
  id, from_store_id → Store, to_store_id → Store
  status: enum(pending | in_transit | received | cancelled)
  notes, created_by → User, created_at
  → StockTransferItem(transfer_id, product_id, qty)

PurchaseOrder
  id, store_id, supplier_id → Supplier
  status: enum(draft | ordered | partial | received | cancelled)
  total_amount (Ks), notes
  created_by → User, created_at
  → PurchaseOrderItem(po_id, product_id, qty, cost_price)

Supplier
  id, name, contact_name, email, phone, address
```

### Sales (Per-Store)

```
Shift
  id, store_id, user_id → User
  opened_at, closed_at
  opening_cash, closing_cash (Ks)
  notes

Order
  id, store_id, shift_id → Shift, cashier_id → User
  order_number (store-scoped sequential, e.g. "ORD-0042")
  status: enum(pending | completed | refunded | held | cancelled)
  subtotal, discount_amount, tax_amount, total  (Ks)
  payment_method: enum(cash | card)
  note, created_at
  → OrderItem(order_id, product_id, product_name*, qty, unit_price, discount, total)
    *product_name snapshot at time of sale

Refund
  id, order_id → Order, store_id, cashier_id → User
  reason, total (Ks), created_at
  → RefundItem(refund_id, order_item_id, qty, amount)
```

### System

```
TaxRate  [per-store]
  id, store_id, name, rate (%), is_default, is_active

ReceiptTemplate  [per-store]
  store_id, header_text, footer_text, show_logo

Device  [per-store]
  id, store_id, name, type, last_seen_at, is_active

AuditLog  [per-store]
  id, store_id, user_id → User
  action, resource, resource_id
  old_value (JSONB), new_value (JSONB)
  ip_address, created_at
```

**Key invariants:**
- `stock_qty` on `StoreProduct` is a cached value updated atomically in the same transaction as each `StockMovement` write — never mutated in isolation
- `OrderItem.product_name` snapshots the name at sale time — history stays accurate after renames
- `order_number` is a sequential integer per store, never resets (e.g. Store A: ORD-0001, ORD-0002 …)

---

## 7. Authentication & RBAC

### Auth Flow

```
POST /auth/login → { accessToken (15min JWT), refreshToken (7d) }
refreshToken stored in Redis: key = userId:deviceId
POST /auth/refresh → rotate refreshToken, issue new accessToken
DELETE /auth/logout → invalidate refreshToken in Redis
```

### Roles

| Role | Scope | Access |
|---|---|---|
| `superadmin` | System-wide | Everything + cross-store data, store management |
| `admin` | Per-store | Full store config, user management, all reports |
| `manager` | Per-store | Reports, stock management, refunds, shift oversight |
| `cashier` | Per-store | POS selling, own shift, own order history |
| `inventory` | Per-store | Stock panel only |

### Permission Examples

```
orders:create       → cashier+
orders:refund       → manager+
products:write      → inventory+
reports:view        → manager+
stores:manage       → superadmin only
users:manage        → admin+
audit_logs:view     → admin+
```

The NestJS `@Roles()` guard resolves `store_id` from the JWT claim and verifies `UserStoreRole` on every request. A token scoped to Store A is automatically rejected for Store B endpoints.

### Frontend Route Protection

| Route group | Required role |
|---|---|
| `/(pos)/` | `cashier` or higher |
| `/(stock)/` | `inventory` or higher |
| `/(dashboard)/` | `manager` or higher |
| `/(admin)/` | `admin` or `superadmin` |

Sidebar nav items are filtered client-side from the decoded JWT role.

---

## 8. API Design

### Module → Route Summary

| Module | Method | Path | Min Role |
|---|---|---|---|
| Auth | POST | `/auth/login` | public |
| Auth | POST | `/auth/refresh` | public |
| Auth | DELETE | `/auth/logout` | any |
| Products | GET | `/products` | any |
| Products | POST | `/products` | inventory+ |
| Products | PATCH | `/products/:id` | inventory+ |
| Products | DELETE | `/products/:id` | admin+ |
| Inventory | GET | `/stores/:storeId/stock` | inventory+ |
| Inventory | POST | `/stores/:storeId/adjustments` | inventory+ |
| Transfers | POST | `/transfers` | manager+ |
| Transfers | PATCH | `/transfers/:id/receive` | inventory+ |
| Purchase Orders | POST | `/purchase-orders` | inventory+ |
| Purchase Orders | PATCH | `/purchase-orders/:id/receive` | inventory+ |
| Shifts | POST | `/shifts/open` | cashier+ |
| Shifts | PATCH | `/shifts/:id/close` | cashier+ |
| Orders | POST | `/orders` | cashier+ |
| Orders | GET | `/orders` | cashier+ |
| Orders | PATCH | `/orders/:id/hold` | cashier+ |
| Orders | PATCH | `/orders/:id/resume` | cashier+ |
| Refunds | POST | `/refunds` | manager+ |
| Reports | GET | `/reports/sales` | manager+ |
| Reports | GET | `/reports/products` | manager+ |
| Reports | GET | `/reports/staff` | manager+ |
| Reports | GET | `/reports/refunds` | manager+ |
| Uploads | POST | `/uploads/image` | inventory+ |
| Devices | GET | `/devices` | admin+ |
| Audit Logs | GET | `/audit-logs` | admin+ |
| Users | GET | `/users` | admin+ |
| Users | POST | `/users` | admin+ |
| Users | PATCH | `/users/:id/roles` | admin+ |
| Stores | GET | `/stores` | superadmin |
| Stores | POST | `/stores` | superadmin |

All store-scoped routes resolve `storeId` from the JWT claim — never from the request body.

### Standard Response Envelope

```typescript
{
  success: boolean
  data: T | null
  error: string | null
  meta?: { total: number; page: number; limit: number }
}
```

---

## 9. Real-time (WebSocket)

Rooms scoped by `storeId`. A terminal in Store A never receives Store B events.

| Event | Direction | Payload | Consumers |
|---|---|---|---|
| `shift:opened` | server → room | `{ shiftId, userId, storeId }` | Dashboard, other POS terminals |
| `shift:closed` | server → room | `{ shiftId, summary }` | Dashboard |
| `order:completed` | server → room | `{ orderId, total, items[] }` | Dashboard live feed |
| `stock:low` | server → room | `{ productId, storeId, qty }` | Stock panel, Dashboard alert |
| `stock:updated` | server → room | `{ productId, storeId, newQty }` | All POS terminals in store |
| `transfer:received` | server → room | `{ transferId, storeId }` | Stock panel |

### Degraded Offline (POS)

When connection drops:
- Cart remains fully usable (products in memory)
- Search falls back to IndexedDB-cached product list
- **Pay button disabled** — banner: "Connection lost — payment unavailable"
- On reconnect: stock levels silently refresh, banner clears

---

## 10. Product Images

- Up to 5 images per product
- First uploaded = main display image; any thumbnail can be promoted to main
- Stored on S3-compatible storage (MinIO locally, AWS S3 in production)
- Auto-generated sizes: 64px (POS grid thumb), 256px (detail view), 800px (full)
- Shown in: POS product grid, stock panel, digital receipt
- Upload: drag & drop or click; max 2MB per image; JPG/PNG/WEBP
- Add/Edit Product opens as a **slide-over drawer** (no page navigation)

---

## 11. Four Panels

### 11.1 Cashier POS

Full-screen layout — no sidebar. Optimised for speed and touch.

**Screens:** Login → Open Shift → POS Main (product grid + cart) → Payment Modal → Receipt Preview → Hold Orders List → Order History → Refund Screen → Close Shift

**Keyboard shortcuts:**
- `F2` — focus search bar
- `F4` — hold order
- `F5` — open payment modal
- `Esc` — clear cart / close modal

**POS Main layout:**
- Top bar: logo, search/barcode input, Hold, History, user menu
- Category chips row (horizontal scroll)
- Product grid: fluid columns (4 on desktop, 3 on tablet portrait, 2 on mobile)
- Right panel: cart, qty controls, summary, Pay + Hold buttons

**Payment flow:** select Cash or Card → enter amount tendered (cash) or confirm (card) → complete → show receipt preview → print/dismiss → cart clears

### 11.2 Dashboard

Dark sidebar + light content. Manager/owner view.

**Pages:** Overview, Sales Analytics, Profit Report, Product Performance, Staff Activity, Refund Report, Device Activity, Low Stock Alerts, Payment Summary, Export Reports

**UI patterns:** KPI cards with delta %, bar/line/donut charts (Recharts), date range filter, store selector (for superadmin), data tables with search/filter/export

### 11.3 Admin Panel

Dark sidebar + light content. System control.

**Pages:** User Management, Roles & Permissions, Store Settings, Tax Settings, Receipt Settings, Payment Settings, Device/Session Management, Audit Logs, System Logs, Notification Settings

### 11.4 Stock Panel

Dark sidebar + light content. Inventory team.

**Pages:** Product List, Add/Edit Product (slide-over), Categories/Brands/Units, Inventory Overview, Stock Movement History, Stock Adjustment, Purchase Orders, Suppliers, Receive Stock, Low Stock, Transfer Stock, Stock Count

---

## 12. Shared Frontend Features

Used across all panels:
- Role-filtered sidebar/top nav
- Search / filter / sort on all tables
- Pagination (cursor-based on large datasets)
- Modals (centred sheet on tablet+, full-screen on mobile)
- Slide-over drawers for create/edit forms
- Toast notifications (success, error, warning, info)
- Loading skeletons
- Empty states with CTAs
- Error boundaries
- Dark mode (optional, not in Phase 1)
- Responsive breakpoints: mobile 375px, tablet 768px, desktop 1280px

---

## 13. Build Order

```
Phase 1 — Foundation
  Monorepo (Turborepo + NestJS + Next.js) scaffolding
  PostgreSQL schema + Prisma migrations
  Auth module (JWT + Redis + refresh rotation)
  Shared design system (tokens, shadcn/ui setup, layout shells)
  Role guard + store-scoping middleware

Phase 2 — Stock & Products
  Product catalog CRUD + image upload
  Categories / brands / units
  StoreProduct + stock levels
  Stock movement ledger

Phase 3 — POS Selling
  Shift open/close
  POS main screen (product grid + cart)
  Order creation + real-time stock deduction
  Hold/resume orders
  Payment modal + digital receipt

Phase 4 — Refunds & History
  Order history screen
  Refund flow
  Cashier shift summary

Phase 5 — Dashboard
  KPI aggregations
  Sales, profit, product performance, staff, refund reports
  Export (CSV + PDF)

Phase 6 — Admin Panel
  User management + role assignment
  Store settings / tax / receipt config
  Audit logs + device management

Phase 7 — Advanced Inventory
  Purchase orders + supplier management
  Stock transfers (cross-store)
  Stock count / cycle count
  Low stock alerts (WebSocket + BullMQ email job)
```

---

## 14. Decisions Log

| Decision | Rationale |
|---|---|
| Monorepo (Turborepo) | Shared types prevent frontend/backend drift across 4 panels |
| Products global, stock per-store | Shared catalog reduces duplication; StoreProduct bridges per-store levels |
| StockMovement append-only ledger | Full audit trail; stock_qty always derivable; no silent mutations |
| OrderItem snapshots product_name | Refunds and history stay accurate after product renames |
| JWT 15min access + 7d refresh in Redis | Short-lived tokens + server-side revocation capability |
| Socket.IO rooms scoped by storeId | Prevents cross-store data leakage in real-time events |
| Degraded offline (read-only) | Avoids complex conflict resolution while protecting against mid-shift outage |
| Cash + card manual only | No hardware integration complexity; matches confirmed payment requirement |
| Currency: Ks (Myanmar Kyat, integer) | No floating point errors; all amounts stored as integer Ks |
| Digital receipts only | No printer SDK complexity; covers primary use case |
| Image upload: up to 5 per product | Practical limit; auto-resized to 3 sizes for performance |
