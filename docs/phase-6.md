# Phase 6 — Admin Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the admin panel — user management, role assignment, store settings, tax rates, receipt template, and audit logs.

**Architecture:** Three new NestJS modules: `UsersModule` (CRUD + role assignment), `StoresModule` (superadmin store management), and `AdminSettingsModule` (tax rates, receipt template, devices). An `AuditLogInterceptor` auto-records mutating requests. All admin routes require `admin+` role; store management requires `superadmin`.

**Tech Stack:** NestJS modules (users, stores, admin-settings), Prisma, Next.js admin panel, React Hook Form + Zod, TanStack Table

**Depends on:** Phase 1 complete (auth, RBAC, Prisma)

---

## File Map

```
apps/api/src/
├── users/
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   └── dto/
│       ├── create-user.dto.ts
│       └── update-user-roles.dto.ts
├── stores/
│   ├── stores.module.ts
│   ├── stores.controller.ts
│   └── stores.service.ts
├── admin-settings/
│   ├── admin-settings.module.ts
│   ├── admin-settings.controller.ts
│   └── admin-settings.service.ts
├── audit-logs/
│   ├── audit-logs.module.ts
│   ├── audit-logs.controller.ts
│   ├── audit-logs.service.ts
│   └── audit-log.interceptor.ts
└── prisma/schema.prisma                       # add TaxRate, ReceiptTemplate, Device, AuditLog

apps/web/src/app/(admin)/
├── admin/users/page.tsx
├── admin/users/_components/
│   ├── UserTable.tsx
│   └── UserDrawer.tsx
├── admin/stores/page.tsx
├── admin/settings/
│   ├── tax/page.tsx
│   └── receipt/page.tsx
└── admin/audit-logs/page.tsx
```

---

## Task 1: Extend Prisma schema — TaxRate, ReceiptTemplate, Device, AuditLog

- [ ] **Step 1: Append models to schema.prisma**

```prisma
model TaxRate {
  id        String  @id @default(cuid())
  storeId   String  @map("store_id")
  name      String
  rate      Float
  isDefault Boolean @default(false) @map("is_default")
  isActive  Boolean @default(true) @map("is_active")

  store     Store   @relation(fields: [storeId], references: [id])

  @@map("tax_rates")
}

model ReceiptTemplate {
  id          String  @id @default(cuid())
  storeId     String  @unique @map("store_id")
  headerText  String? @map("header_text")
  footerText  String? @map("footer_text")
  showLogo    Boolean @default(true) @map("show_logo")

  store       Store   @relation(fields: [storeId], references: [id])

  @@map("receipt_templates")
}

model Device {
  id         String   @id @default(cuid())
  storeId    String   @map("store_id")
  name       String
  type       String
  lastSeenAt DateTime? @map("last_seen_at")
  isActive   Boolean  @default(true) @map("is_active")

  store      Store    @relation(fields: [storeId], references: [id])

  @@map("devices")
}

model AuditLog {
  id          String   @id @default(cuid())
  storeId     String?  @map("store_id")
  userId      String?  @map("user_id")
  action      String
  resource    String
  resourceId  String?  @map("resource_id")
  oldValue    Json?    @map("old_value")
  newValue    Json?    @map("new_value")
  ipAddress   String?  @map("ip_address")
  createdAt   DateTime @default(now()) @map("created_at")

  user        User?    @relation(fields: [userId], references: [id])

  @@map("audit_logs")
}
```

- [ ] **Step 2: Run migration**

```bash
cd apps/api && npx prisma migrate dev --name add_admin_tables
```

Expected: Migration applied successfully.

- [ ] **Step 3: Commit**

```bash
git add apps/api/prisma/
git commit -m "feat: add TaxRate, ReceiptTemplate, Device, AuditLog to Prisma schema"
```

---

## Task 2: Users module (NestJS)

- [ ] **Step 1: Create create-user.dto.ts**

```typescript
// apps/api/src/users/dto/create-user.dto.ts
import { IsEmail, IsNotEmpty, MinLength, IsEnum, IsOptional } from 'class-validator'
import { Role } from '@prisma/client'

export class CreateUserDto {
  @IsNotEmpty() name: string
  @IsEmail() email: string
  @MinLength(8) password: string
  @IsEnum(Role) role: Role
  @IsOptional() storeId?: string
}
```

- [ ] **Step 2: Create update-user-roles.dto.ts**

```typescript
// apps/api/src/users/dto/update-user-roles.dto.ts
import { IsEnum, IsNotEmpty } from 'class-validator'
import { Role } from '@prisma/client'

export class UpdateUserRolesDto {
  @IsNotEmpty() storeId: string
  @IsEnum(Role) role: Role
}
```

- [ ] **Step 3: Create users.service.ts**

```typescript
// apps/api/src/users/users.service.ts
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import * as bcrypt from 'bcrypt'
import { CreateUserDto } from './dto/create-user.dto'
import { UpdateUserRolesDto } from './dto/update-user-roles.dto'

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async createUser(dto: CreateUserDto, requestingStoreId: string) {
    const existing = await this.prisma.user.findUnique({ where: { email: dto.email } })
    if (existing) throw new ConflictException('Email already in use')

    const passwordHash = await bcrypt.hash(dto.password, 12)
    const user = await this.prisma.user.create({
      data: { name: dto.name, email: dto.email, passwordHash },
    })

    const storeId = dto.storeId ?? requestingStoreId
    await this.prisma.userStoreRole.create({
      data: { userId: user.id, storeId, role: dto.role },
    })

    return this.prisma.user.findUnique({
      where: { id: user.id },
      include: { storeRoles: true },
    })
  }

  async getUsers(storeId: string) {
    return this.prisma.user.findMany({
      where: { storeRoles: { some: { storeId } } },
      include: { storeRoles: { where: { storeId } } },
      orderBy: { name: 'asc' },
    })
  }

  async updateRoles(userId: string, dto: UpdateUserRolesDto) {
    const user = await this.prisma.user.findUnique({ where: { id: userId } })
    if (!user) throw new NotFoundException('User not found')

    return this.prisma.userStoreRole.upsert({
      where: { userId_storeId: { userId, storeId: dto.storeId } },
      create: { userId, storeId: dto.storeId, role: dto.role },
      update: { role: dto.role },
    })
  }

  async deactivateUser(userId: string) {
    const user = await this.prisma.user.findUnique({ where: { id: userId } })
    if (!user) throw new NotFoundException('User not found')
    return this.prisma.user.update({ where: { id: userId }, data: { isActive: false } })
  }
}
```

- [ ] **Step 4: Create users.controller.ts**

```typescript
// apps/api/src/users/users.controller.ts
import { Controller, Get, Post, Patch, Param, Body, Request, UseGuards } from '@nestjs/common'
import { UsersService } from './users.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'
import { CreateUserDto } from './dto/create-user.dto'
import { UpdateUserRolesDto } from './dto/update-user-roles.dto'

@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @Roles('ADMIN')
  getUsers(@Request() req: any) {
    return this.usersService.getUsers(req.user.storeId)
  }

  @Post()
  @Roles('ADMIN')
  createUser(@Body() dto: CreateUserDto, @Request() req: any) {
    return this.usersService.createUser(dto, req.user.storeId)
  }

  @Patch(':id/roles')
  @Roles('ADMIN')
  updateRoles(@Param('id') id: string, @Body() dto: UpdateUserRolesDto) {
    return this.usersService.updateRoles(id, dto)
  }

  @Patch(':id/deactivate')
  @Roles('ADMIN')
  deactivate(@Param('id') id: string) {
    return this.usersService.deactivateUser(id)
  }
}
```

- [ ] **Step 5: Create users.module.ts and register in AppModule**

```typescript
// apps/api/src/users/users.module.ts
import { Module } from '@nestjs/common'
import { UsersController } from './users.controller'
import { UsersService } from './users.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Add `UsersModule` to `apps/api/src/app.module.ts` imports.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/users/
git commit -m "feat: add users module (CRUD + role assignment)"
```

---

## Task 3: Stores module (NestJS — superadmin only)

- [ ] **Step 1: Create stores.service.ts**

```typescript
// apps/api/src/stores/stores.service.ts
import { Injectable, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class StoresService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.store.findMany({ orderBy: { name: 'asc' } })
  }

  async findOne(id: string) {
    const store = await this.prisma.store.findUnique({ where: { id } })
    if (!store) throw new NotFoundException('Store not found')
    return store
  }

  create(dto: { name: string; address?: string; phone?: string; timezone?: string }) {
    return this.prisma.store.create({ data: { ...dto } })
  }

  async update(id: string, dto: Partial<{ name: string; address: string; phone: string; isActive: boolean }>) {
    await this.findOne(id)
    return this.prisma.store.update({ where: { id }, data: dto })
  }
}
```

- [ ] **Step 2: Create stores.controller.ts**

```typescript
// apps/api/src/stores/stores.controller.ts
import { Controller, Get, Post, Patch, Param, Body, UseGuards } from '@nestjs/common'
import { StoresService } from './stores.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('stores')
@UseGuards(JwtAuthGuard, RolesGuard)
export class StoresController {
  constructor(private readonly storesService: StoresService) {}

  @Get()
  @Roles('SUPERADMIN')
  findAll() { return this.storesService.findAll() }

  @Post()
  @Roles('SUPERADMIN')
  create(@Body() body: any) { return this.storesService.create(body) }

  @Patch(':id')
  @Roles('SUPERADMIN')
  update(@Param('id') id: string, @Body() body: any) { return this.storesService.update(id, body) }
}
```

- [ ] **Step 3: Create stores.module.ts and register in AppModule**

```typescript
// apps/api/src/stores/stores.module.ts
import { Module } from '@nestjs/common'
import { StoresController } from './stores.controller'
import { StoresService } from './stores.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [StoresController],
  providers: [StoresService],
})
export class StoresModule {}
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/stores/
git commit -m "feat: add stores module (superadmin)"
```

---

## Task 4: Admin settings module (tax rates, receipt template)

- [ ] **Step 1: Create admin-settings.service.ts**

```typescript
// apps/api/src/admin-settings/admin-settings.service.ts
import { Injectable } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class AdminSettingsService {
  constructor(private readonly prisma: PrismaService) {}

  // Tax rates
  getTaxRates(storeId: string) {
    return this.prisma.taxRate.findMany({ where: { storeId }, orderBy: { name: 'asc' } })
  }

  createTaxRate(storeId: string, dto: { name: string; rate: number; isDefault?: boolean }) {
    return this.prisma.taxRate.create({ data: { storeId, ...dto } })
  }

  updateTaxRate(id: string, dto: Partial<{ name: string; rate: number; isDefault: boolean; isActive: boolean }>) {
    return this.prisma.taxRate.update({ where: { id }, data: dto })
  }

  deleteTaxRate(id: string) {
    return this.prisma.taxRate.delete({ where: { id } })
  }

  // Receipt template
  async getReceiptTemplate(storeId: string) {
    return (
      (await this.prisma.receiptTemplate.findUnique({ where: { storeId } })) ??
      { storeId, headerText: null, footerText: null, showLogo: true }
    )
  }

  upsertReceiptTemplate(
    storeId: string,
    dto: { headerText?: string; footerText?: string; showLogo?: boolean },
  ) {
    return this.prisma.receiptTemplate.upsert({
      where: { storeId },
      create: { storeId, ...dto },
      update: dto,
    })
  }

  // Devices
  getDevices(storeId: string) {
    return this.prisma.device.findMany({ where: { storeId }, orderBy: { name: 'asc' } })
  }
}
```

- [ ] **Step 2: Create admin-settings.controller.ts**

```typescript
// apps/api/src/admin-settings/admin-settings.controller.ts
import { Controller, Get, Post, Patch, Delete, Param, Body, Request, UseGuards } from '@nestjs/common'
import { AdminSettingsService } from './admin-settings.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('settings')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminSettingsController {
  constructor(private readonly service: AdminSettingsService) {}

  @Get('tax-rates')
  @Roles('ADMIN')
  getTaxRates(@Request() req: any) { return this.service.getTaxRates(req.user.storeId) }

  @Post('tax-rates')
  @Roles('ADMIN')
  createTaxRate(@Body() body: any, @Request() req: any) {
    return this.service.createTaxRate(req.user.storeId, body)
  }

  @Patch('tax-rates/:id')
  @Roles('ADMIN')
  updateTaxRate(@Param('id') id: string, @Body() body: any) {
    return this.service.updateTaxRate(id, body)
  }

  @Delete('tax-rates/:id')
  @Roles('ADMIN')
  deleteTaxRate(@Param('id') id: string) { return this.service.deleteTaxRate(id) }

  @Get('receipt')
  @Roles('ADMIN')
  getReceipt(@Request() req: any) { return this.service.getReceiptTemplate(req.user.storeId) }

  @Patch('receipt')
  @Roles('ADMIN')
  updateReceipt(@Body() body: any, @Request() req: any) {
    return this.service.upsertReceiptTemplate(req.user.storeId, body)
  }

  @Get('devices')
  @Roles('ADMIN')
  getDevices(@Request() req: any) { return this.service.getDevices(req.user.storeId) }
}
```

- [ ] **Step 3: Create admin-settings.module.ts and register in AppModule**

```typescript
// apps/api/src/admin-settings/admin-settings.module.ts
import { Module } from '@nestjs/common'
import { AdminSettingsController } from './admin-settings.controller'
import { AdminSettingsService } from './admin-settings.service'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [AdminSettingsController],
  providers: [AdminSettingsService],
})
export class AdminSettingsModule {}
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/admin-settings/
git commit -m "feat: add admin-settings module (tax rates, receipt template, devices)"
```

---

## Task 5: Audit log interceptor + module

- [ ] **Step 1: Create audit-log.interceptor.ts**

```typescript
// apps/api/src/audit-logs/audit-log.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common'
import { Observable, tap } from 'rxjs'
import { AuditLogsService } from './audit-logs.service'

const MUTATING_METHODS = new Set(['POST', 'PATCH', 'PUT', 'DELETE'])

@Injectable()
export class AuditLogInterceptor implements NestInterceptor {
  constructor(private readonly auditLogsService: AuditLogsService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const req = context.switchToHttp().getRequest()
    if (!MUTATING_METHODS.has(req.method)) return next.handle()

    const user = req.user
    const action = req.method
    const resource = req.url.split('/')[1] ?? 'unknown'

    return next.handle().pipe(
      tap((data) => {
        if (!user) return
        this.auditLogsService
          .log({
            storeId: user.storeId ?? null,
            userId: user.sub,
            action,
            resource,
            newValue: data ?? null,
            ipAddress: req.ip,
          })
          .catch(() => undefined) // non-blocking
      }),
    )
  }
}
```

- [ ] **Step 2: Create audit-logs.service.ts**

```typescript
// apps/api/src/audit-logs/audit-logs.service.ts
import { Injectable } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

interface LogDto {
  storeId: string | null
  userId: string
  action: string
  resource: string
  resourceId?: string
  oldValue?: unknown
  newValue?: unknown
  ipAddress?: string
}

@Injectable()
export class AuditLogsService {
  constructor(private readonly prisma: PrismaService) {}

  log(dto: LogDto) {
    return this.prisma.auditLog.create({
      data: {
        storeId: dto.storeId,
        userId: dto.userId,
        action: dto.action,
        resource: dto.resource,
        resourceId: dto.resourceId ?? null,
        oldValue: dto.oldValue ?? undefined,
        newValue: dto.newValue ?? undefined,
        ipAddress: dto.ipAddress ?? null,
      },
    })
  }

  getLogs(storeId: string, page = 1, limit = 50) {
    const skip = (page - 1) * limit
    return this.prisma.auditLog.findMany({
      where: { storeId },
      include: { user: { select: { id: true, name: true } } },
      orderBy: { createdAt: 'desc' },
      skip,
      take: limit,
    })
  }
}
```

- [ ] **Step 3: Create audit-logs.controller.ts**

```typescript
// apps/api/src/audit-logs/audit-logs.controller.ts
import { Controller, Get, Query, Request, UseGuards } from '@nestjs/common'
import { AuditLogsService } from './audit-logs.service'
import { JwtAuthGuard } from '../auth/jwt-auth.guard'
import { RolesGuard } from '../auth/roles.guard'
import { Roles } from '../auth/roles.decorator'

@Controller('audit-logs')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AuditLogsController {
  constructor(private readonly service: AuditLogsService) {}

  @Get()
  @Roles('ADMIN')
  getLogs(@Request() req: any, @Query('page') page?: string) {
    return this.service.getLogs(req.user.storeId, page ? parseInt(page) : 1)
  }
}
```

- [ ] **Step 4: Create audit-logs.module.ts and register in AppModule**

```typescript
// apps/api/src/audit-logs/audit-logs.module.ts
import { Module } from '@nestjs/common'
import { AuditLogsController } from './audit-logs.controller'
import { AuditLogsService } from './audit-logs.service'
import { AuditLogInterceptor } from './audit-log.interceptor'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [AuditLogsController],
  providers: [AuditLogsService, AuditLogInterceptor],
  exports: [AuditLogsService, AuditLogInterceptor],
})
export class AuditLogsModule {}
```

Register `APP_INTERCEPTOR` with `AuditLogInterceptor` as a global interceptor in `app.module.ts`:

```typescript
// In app.module.ts providers array:
{ provide: APP_INTERCEPTOR, useClass: AuditLogInterceptor }
```

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/audit-logs/
git commit -m "feat: add audit log interceptor and module"
```

---

## Task 6: Admin panel frontend

- [ ] **Step 1: Create user management page**

```tsx
// apps/web/src/app/(admin)/admin/users/page.tsx
'use client'
import { useState } from 'react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'
import { Plus } from 'lucide-react'
import { UserDrawer } from './_components/UserDrawer'

export default function UsersPage() {
  const [drawerOpen, setDrawerOpen] = useState(false)
  const queryClient = useQueryClient()

  const { data: users, isLoading } = useQuery({
    queryKey: ['admin-users'],
    queryFn: () => apiFetch('/users').then((r) => r.data),
  })

  const deactivateMutation = useMutation({
    mutationFn: (userId: string) =>
      apiFetch(`/users/${userId}/deactivate`, { method: 'PATCH' }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['admin-users'] }),
  })

  const ROLE_BADGE: Record<string, string> = {
    SUPERADMIN: 'bg-danger/10 text-danger',
    ADMIN: 'bg-primary/10 text-primary',
    MANAGER: 'bg-info/10 text-info',
    CASHIER: 'bg-canvas text-text-muted border border-border',
    INVENTORY: 'bg-canvas text-text-muted border border-border',
  }

  return (
    <div className="p-6">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-lg font-bold text-text-dark">Users</h1>
        <button
          onClick={() => setDrawerOpen(true)}
          className="flex items-center gap-2 px-4 py-2 bg-primary text-white rounded-xl text-sm font-semibold min-h-[44px]"
        >
          <Plus size={14} /> Add User
        </button>
      </div>

      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['Name', 'Email', 'Role', 'Status', 'Actions'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {isLoading
              ? Array.from({ length: 4 }).map((_, i) => (
                  <tr key={i}>
                    <td colSpan={5} className="px-4 py-3">
                      <div className="h-4 bg-canvas rounded animate-pulse" />
                    </td>
                  </tr>
                ))
              : (users ?? []).map((user: any) => (
                  <tr key={user.id} className="border-b border-border last:border-0 hover:bg-canvas">
                    <td className="px-4 py-3 font-semibold">{user.name}</td>
                    <td className="px-4 py-3 text-text-muted">{user.email}</td>
                    <td className="px-4 py-3">
                      <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${ROLE_BADGE[user.storeRoles?.[0]?.role] ?? ''}`}>
                        {user.storeRoles?.[0]?.role ?? '—'}
                      </span>
                    </td>
                    <td className="px-4 py-3">
                      <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${user.isActive ? 'bg-primary/10 text-primary' : 'bg-canvas text-text-muted border border-border'}`}>
                        {user.isActive ? 'Active' : 'Inactive'}
                      </span>
                    </td>
                    <td className="px-4 py-3">
                      {user.isActive && (
                        <button
                          onClick={() => deactivateMutation.mutate(user.id)}
                          className="text-xs text-danger font-semibold min-h-[36px] px-2"
                        >
                          Deactivate
                        </button>
                      )}
                    </td>
                  </tr>
                ))}
          </tbody>
        </table>
      </div>

      {drawerOpen && (
        <UserDrawer
          onClose={() => setDrawerOpen(false)}
          onSuccess={() => {
            setDrawerOpen(false)
            queryClient.invalidateQueries({ queryKey: ['admin-users'] })
          }}
        />
      )}
    </div>
  )
}
```

- [ ] **Step 2: Create UserDrawer.tsx**

```tsx
// apps/web/src/app/(admin)/admin/users/_components/UserDrawer.tsx
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { apiFetch } from '@/lib/api'

const schema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  password: z.string().min(8),
  role: z.enum(['ADMIN', 'MANAGER', 'CASHIER', 'INVENTORY']),
})

type FormValues = z.infer<typeof schema>

interface Props { onClose: () => void; onSuccess: () => void }

export function UserDrawer({ onClose, onSuccess }: Props) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormValues>({
    resolver: zodResolver(schema),
  })

  async function onSubmit(values: FormValues) {
    await apiFetch('/users', { method: 'POST', body: JSON.stringify(values) })
    onSuccess()
  }

  return (
    <div className="fixed inset-0 z-40 flex">
      <div className="flex-1 bg-black/30" onClick={onClose} />
      <div className="w-full max-w-md bg-surface border-l border-border flex flex-col">
        <div className="flex items-center justify-between p-5 border-b border-border bg-white">
          <h2 className="text-sm font-bold text-text-dark">Add User</h2>
          <button onClick={onClose} className="w-8 h-8 flex items-center justify-center rounded-lg bg-canvas text-text-muted text-lg">✕</button>
        </div>

        <form onSubmit={handleSubmit(onSubmit)} className="flex-1 overflow-y-auto p-5 space-y-4">
          {(['name', 'email', 'password'] as const).map((field) => (
            <div key={field}>
              <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">
                {field.charAt(0).toUpperCase() + field.slice(1)}
              </label>
              <input
                {...register(field)}
                type={field === 'password' ? 'password' : 'text'}
                className="w-full px-4 py-2.5 border border-border rounded-xl text-sm focus:outline-none focus:border-primary min-h-[44px]"
                style={{ fontSize: '16px' }}
              />
              {errors[field] && <p className="text-xs text-danger mt-1">{errors[field]?.message}</p>}
            </div>
          ))}

          <div>
            <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">Role</label>
            <select
              {...register('role')}
              className="w-full px-4 py-2.5 border border-border rounded-xl text-sm focus:outline-none focus:border-primary min-h-[44px] bg-white"
            >
              {['ADMIN', 'MANAGER', 'CASHIER', 'INVENTORY'].map((r) => (
                <option key={r} value={r}>{r}</option>
              ))}
            </select>
          </div>
        </form>

        <div className="p-5 border-t border-border bg-white flex gap-3">
          <button onClick={onClose} className="flex-1 py-3 border border-border rounded-xl text-sm font-semibold min-h-[44px]">Cancel</button>
          <button
            onClick={handleSubmit(onSubmit)}
            disabled={isSubmitting}
            className="flex-1 py-3 bg-primary text-white rounded-xl text-sm font-bold min-h-[44px] disabled:opacity-40"
          >
            {isSubmitting ? 'Saving…' : 'Save User'}
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create tax settings page**

```tsx
// apps/web/src/app/(admin)/admin/settings/tax/page.tsx
'use client'
import { useState } from 'react'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'
import { Plus, Trash2 } from 'lucide-react'

export default function TaxPage() {
  const [name, setName] = useState('')
  const [rate, setRate] = useState('')
  const qc = useQueryClient()

  const { data: rates } = useQuery({
    queryKey: ['tax-rates'],
    queryFn: () => apiFetch('/settings/tax-rates').then((r) => r.data),
  })

  const createMutation = useMutation({
    mutationFn: () => apiFetch('/settings/tax-rates', {
      method: 'POST',
      body: JSON.stringify({ name, rate: parseFloat(rate) }),
    }),
    onSuccess: () => { qc.invalidateQueries({ queryKey: ['tax-rates'] }); setName(''); setRate('') },
  })

  const deleteMutation = useMutation({
    mutationFn: (id: string) => apiFetch(`/settings/tax-rates/${id}`, { method: 'DELETE' }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['tax-rates'] }),
  })

  return (
    <div className="p-6 max-w-lg">
      <h1 className="text-lg font-bold text-text-dark mb-6">Tax Rates</h1>

      <div className="space-y-2 mb-6">
        {(rates ?? []).map((r: any) => (
          <div key={r.id} className="bg-surface border border-border rounded-xl p-4 flex items-center justify-between">
            <div>
              <p className="text-sm font-semibold">{r.name}</p>
              <p className="text-xs text-text-muted">{r.rate}%</p>
            </div>
            <button
              onClick={() => deleteMutation.mutate(r.id)}
              className="text-danger p-2 min-h-[44px] min-w-[44px] flex items-center justify-center"
            >
              <Trash2 size={14} />
            </button>
          </div>
        ))}
      </div>

      <div className="bg-surface border border-border rounded-xl p-4 space-y-3">
        <p className="text-xs font-bold uppercase tracking-wide text-text-muted">Add Tax Rate</p>
        <div className="flex gap-3">
          <input
            value={name}
            onChange={(e) => setName(e.target.value)}
            placeholder="Name (e.g. VAT)"
            className="flex-1 px-3 py-2.5 border border-border rounded-xl text-sm focus:outline-none focus:border-primary min-h-[44px]"
            style={{ fontSize: '16px' }}
          />
          <input
            type="number"
            value={rate}
            onChange={(e) => setRate(e.target.value)}
            placeholder="%"
            className="w-20 px-3 py-2.5 border border-border rounded-xl text-sm focus:outline-none focus:border-primary min-h-[44px]"
            style={{ fontSize: '16px' }}
          />
          <button
            onClick={() => createMutation.mutate()}
            disabled={!name || !rate}
            className="px-4 bg-primary text-white rounded-xl min-h-[44px] disabled:opacity-40"
          >
            <Plus size={16} />
          </button>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create receipt settings page**

```tsx
// apps/web/src/app/(admin)/admin/settings/receipt/page.tsx
'use client'
import { useEffect } from 'react'
import { useForm } from 'react-hook-form'
import { useQuery, useMutation } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'

interface FormValues {
  headerText: string
  footerText: string
  showLogo: boolean
}

export default function ReceiptSettingsPage() {
  const { data } = useQuery({
    queryKey: ['receipt-template'],
    queryFn: () => apiFetch('/settings/receipt').then((r) => r.data),
  })

  const { register, handleSubmit, reset, formState: { isSubmitting } } = useForm<FormValues>()

  useEffect(() => { if (data) reset(data) }, [data, reset])

  const saveMutation = useMutation({
    mutationFn: (values: FormValues) =>
      apiFetch('/settings/receipt', { method: 'PATCH', body: JSON.stringify(values) }),
  })

  return (
    <div className="p-6 max-w-lg">
      <h1 className="text-lg font-bold text-text-dark mb-6">Receipt Settings</h1>
      <form onSubmit={handleSubmit((v) => saveMutation.mutate(v))} className="space-y-4">
        <div>
          <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">Header Text</label>
          <textarea {...register('headerText')} rows={2} className="w-full px-4 py-2.5 border border-border rounded-xl text-sm focus:outline-none focus:border-primary resize-none" />
        </div>
        <div>
          <label className="text-xs font-bold uppercase tracking-wide text-text-muted block mb-1">Footer Text</label>
          <textarea {...register('footerText')} rows={2} className="w-full px-4 py-2.5 border border-border rounded-xl text-sm focus:outline-none focus:border-primary resize-none" />
        </div>
        <div className="flex items-center gap-3 p-4 bg-canvas border border-border rounded-xl">
          <input type="checkbox" {...register('showLogo')} className="w-5 h-5 accent-primary" />
          <div>
            <p className="text-sm font-semibold">Show Store Logo</p>
            <p className="text-xs text-text-muted">Display logo on printed receipt</p>
          </div>
        </div>
        <button
          type="submit"
          disabled={isSubmitting}
          className="w-full py-3 bg-primary text-white rounded-xl font-bold text-sm min-h-[44px] disabled:opacity-40"
        >
          {isSubmitting ? 'Saving…' : 'Save Settings'}
        </button>
      </form>
    </div>
  )
}
```

- [ ] **Step 5: Create audit logs page**

```tsx
// apps/web/src/app/(admin)/admin/audit-logs/page.tsx
'use client'
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'

export default function AuditLogsPage() {
  const [page, setPage] = useState(1)

  const { data } = useQuery({
    queryKey: ['audit-logs', page],
    queryFn: () => apiFetch(`/audit-logs?page=${page}`).then((r) => r.data),
  })

  return (
    <div className="p-6">
      <h1 className="text-lg font-bold text-text-dark mb-6">Audit Logs</h1>
      <div className="bg-surface border border-border rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-canvas border-b border-border">
            <tr>
              {['Time', 'User', 'Action', 'Resource'].map((h) => (
                <th key={h} className="text-left px-4 py-3 text-xs font-bold uppercase tracking-wide text-text-muted">
                  {h}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {(data ?? []).map((log: any) => (
              <tr key={log.id} className="border-b border-border last:border-0 hover:bg-canvas">
                <td className="px-4 py-3 text-text-muted text-xs">{new Date(log.createdAt).toLocaleString()}</td>
                <td className="px-4 py-3">{log.user?.name ?? '—'}</td>
                <td className="px-4 py-3">
                  <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${
                    log.action === 'DELETE' ? 'bg-danger/10 text-danger' :
                    log.action === 'POST' ? 'bg-primary/10 text-primary' :
                    'bg-canvas text-text-muted border border-border'
                  }`}>{log.action}</span>
                </td>
                <td className="px-4 py-3 text-text-muted">{log.resource}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      <div className="flex justify-center gap-4 mt-4">
        <button disabled={page === 1} onClick={() => setPage((p) => p - 1)} className="px-4 py-2 border border-border rounded-lg text-sm disabled:opacity-40 min-h-[44px]">Previous</button>
        <button onClick={() => setPage((p) => p + 1)} className="px-4 py-2 border border-border rounded-lg text-sm min-h-[44px]">Next</button>
      </div>
    </div>
  )
}
```

- [ ] **Step 6: Run all tests**

```bash
cd apps/api && npx jest --no-coverage
```

Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add apps/web/src/app/(admin)/
git commit -m "feat: add admin panel — users, tax, receipt, audit log pages"
```
