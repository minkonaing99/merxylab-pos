# Phase 1 — Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the MerxyLab POS monorepo with working JWT authentication, a shared design system, and role-protected layout shells for all four panels.

**Architecture:** Turborepo monorepo with `apps/api` (NestJS + Fastify), `apps/web` (Next.js 14 App Router), and shared `packages/types` + `packages/utils`. Auth uses JWT access tokens (15 min) + refresh tokens in Redis (7 days) with rotation on every use. All store-scoped endpoints resolve `storeId` from the JWT claim — never from the request body.

**Tech Stack:** pnpm workspaces, Turborepo, NestJS 10, Fastify, Prisma 5, PostgreSQL 16, Redis (ioredis), bcryptjs, Jest, Next.js 14, Tailwind CSS 3, shadcn/ui, Zustand, TanStack Query, axios

---

## File Map

```
merxylab-pos/
├── package.json                              # root workspace
├── pnpm-workspace.yaml
├── turbo.json
├── .gitignore
├── .env.example
│
├── apps/
│   ├── api/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── jest.config.ts
│   │   ├── prisma/
│   │   │   └── schema.prisma                # Phase 1 tables: User, Store, UserStoreRole
│   │   └── src/
│   │       ├── main.ts                       # Fastify bootstrap
│   │       ├── app.module.ts
│   │       ├── common/
│   │       │   ├── types/api-response.type.ts
│   │       │   ├── filters/http-exception.filter.ts
│   │       │   └── interceptors/response.interceptor.ts
│   │       ├── prisma/
│   │       │   ├── prisma.module.ts
│   │       │   └── prisma.service.ts
│   │       ├── redis/
│   │       │   ├── redis.module.ts
│   │       │   └── redis.service.ts
│   │       └── auth/
│   │           ├── auth.module.ts
│   │           ├── auth.controller.ts
│   │           ├── auth.service.ts
│   │           ├── auth.service.spec.ts      # unit tests
│   │           ├── auth.controller.spec.ts   # unit tests
│   │           ├── dto/
│   │           │   ├── login.dto.ts
│   │           │   └── refresh.dto.ts
│   │           ├── strategies/
│   │           │   └── jwt.strategy.ts
│   │           ├── guards/
│   │           │   ├── jwt-auth.guard.ts
│   │           │   └── roles.guard.ts
│   │           └── decorators/
│   │               ├── roles.decorator.ts
│   │               └── current-user.decorator.ts
│   │
│   └── web/
│       ├── package.json
│       ├── tsconfig.json
│       ├── tailwind.config.ts
│       ├── postcss.config.js
│       ├── next.config.ts
│       ├── middleware.ts                     # route protection
│       ├── components.json                   # shadcn/ui config
│       └── src/
│           ├── app/
│           │   ├── layout.tsx                # root layout
│           │   ├── globals.css               # design tokens + base styles
│           │   ├── (auth)/
│           │   │   └── login/
│           │   │       └── page.tsx
│           │   ├── (pos)/
│           │   │   └── layout.tsx            # cashier shell
│           │   ├── (dashboard)/
│           │   │   └── layout.tsx            # manager shell
│           │   ├── (stock)/
│           │   │   └── layout.tsx            # inventory shell
│           │   └── (admin)/
│           │       └── layout.tsx            # admin shell
│           ├── components/
│           │   ├── layout/
│           │   │   ├── AppShell.tsx          # sidebar + main area
│           │   │   ├── Sidebar.tsx           # full sidebar (desktop)
│           │   │   ├── SidebarRail.tsx       # icon-only collapsed (tablet)
│           │   │   ├── SidebarSheet.tsx      # bottom sheet (mobile)
│           │   │   └── TopBar.tsx
│           │   └── ui/                       # shadcn/ui components (generated)
│           ├── lib/
│           │   ├── api.ts                    # axios instance
│           │   └── auth.ts                   # JWT decode helpers
│           └── store/
│               └── auth.store.ts             # Zustand auth store
│
└── packages/
    ├── types/
    │   ├── package.json
    │   ├── tsconfig.json
    │   └── src/
    │       ├── index.ts
    │       ├── enums.ts                      # Role enum
    │       ├── auth.types.ts                 # JwtPayload, AuthUser
    │       ├── store.types.ts                # Store, UserStoreRole
    │       └── api.types.ts                  # ApiResponse<T>
    └── utils/
        ├── package.json
        ├── tsconfig.json
        └── src/
            ├── index.ts
            ├── currency.ts                   # formatKs(amount: number): string
            └── date.ts                       # formatDate helpers
```

---

## Task 1: Initialize monorepo

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `turbo.json`
- Create: `.gitignore`
- Create: `.env.example`

- [ ] **Step 1: Initialize git and connect remote**

```bash
cd /Users/aurora/Documents/structure/03_projects/active/pos
git init
git remote add origin https://github.com/minkonaing99/merxylab-pos.git
```

- [ ] **Step 2: Create root package.json**

```json
{
  "name": "merxylab-pos",
  "private": true,
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "db:migrate": "pnpm --filter api prisma migrate dev",
    "db:generate": "pnpm --filter api prisma generate",
    "db:studio": "pnpm --filter api prisma studio"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.4.0"
  },
  "engines": {
    "node": ">=20",
    "pnpm": ">=9"
  },
  "packageManager": "pnpm@9.0.0"
}
```

- [ ] **Step 3: Create pnpm-workspace.yaml**

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

- [ ] **Step 4: Create turbo.json**

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalEnv": ["DATABASE_URL", "REDIS_URL", "JWT_ACCESS_SECRET", "JWT_REFRESH_SECRET"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "lint": {}
  }
}
```

- [ ] **Step 5: Create .gitignore**

```
node_modules/
.env
.env.local
dist/
.next/
.turbo/
*.tsbuildinfo
coverage/
```

- [ ] **Step 6: Create .env.example**

```env
# Database
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/merxylab_pos"

# Redis
REDIS_URL="redis://localhost:6379"

# JWT
JWT_ACCESS_SECRET="change-me-access-secret-min-32-chars"
JWT_REFRESH_SECRET="change-me-refresh-secret-min-32-chars"
JWT_ACCESS_EXPIRES_IN="15m"
JWT_REFRESH_EXPIRES_IN="7d"

# App
API_PORT=3001
NODE_ENV=development
```

- [ ] **Step 7: Create .env by copying .env.example, then fill in local values**

```bash
cp .env.example .env
```

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "chore: initialize monorepo with Turborepo and pnpm workspaces"
```

---

## Task 2: Scaffold NestJS API

**Files:**
- Create: `apps/api/package.json`
- Create: `apps/api/tsconfig.json`
- Create: `apps/api/jest.config.ts`
- Create: `apps/api/src/main.ts`
- Create: `apps/api/src/app.module.ts`
- Create: `apps/api/src/common/types/api-response.type.ts`
- Create: `apps/api/src/common/filters/http-exception.filter.ts`
- Create: `apps/api/src/common/interceptors/response.interceptor.ts`

- [ ] **Step 1: Create apps/api/package.json**

```json
{
  "name": "api",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/main",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "prisma": "prisma"
  },
  "dependencies": {
    "@nestjs/common": "^10.3.0",
    "@nestjs/config": "^3.2.0",
    "@nestjs/core": "^10.3.0",
    "@nestjs/jwt": "^10.2.0",
    "@nestjs/passport": "^10.0.3",
    "@nestjs/platform-fastify": "^10.3.0",
    "@nestjs/websockets": "^10.3.0",
    "@nestjs/socket.io": "^10.3.0",
    "@prisma/client": "^5.14.0",
    "bcryptjs": "^2.4.3",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "ioredis": "^5.3.2",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.1",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.3.2",
    "@nestjs/schematics": "^10.1.1",
    "@nestjs/testing": "^10.3.0",
    "@types/bcryptjs": "^2.4.6",
    "@types/jest": "^29.5.12",
    "@types/node": "^20.12.0",
    "@types/passport-jwt": "^4.0.1",
    "jest": "^29.7.0",
    "prisma": "^5.14.0",
    "ts-jest": "^29.1.4",
    "ts-node": "^10.9.2",
    "typescript": "^5.4.0"
  }
}
```

- [ ] **Step 2: Create apps/api/tsconfig.json**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false
  }
}
```

- [ ] **Step 3: Create apps/api/jest.config.ts**

```typescript
import type { Config } from 'jest'

const config: Config = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  collectCoverageFrom: ['**/*.(t|j)s'],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
}

export default config
```

- [ ] **Step 4: Create apps/api/src/common/types/api-response.type.ts**

```typescript
export interface ApiMeta {
  total: number
  page: number
  limit: number
}

export interface ApiResponse<T = unknown> {
  success: boolean
  data: T | null
  error: string | null
  meta?: ApiMeta
}

export function ok<T>(data: T, meta?: ApiMeta): ApiResponse<T> {
  return { success: true, data, error: null, ...(meta ? { meta } : {}) }
}

export function fail(error: string): ApiResponse<null> {
  return { success: false, data: null, error }
}
```

- [ ] **Step 5: Create apps/api/src/common/filters/http-exception.filter.ts**

```typescript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
} from '@nestjs/common'
import { FastifyReply } from 'fastify'
import { fail } from '../types/api-response.type'

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp()
    const reply = ctx.getResponse<FastifyReply>()

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR

    const message =
      exception instanceof HttpException
        ? (exception.getResponse() as any)?.message ?? exception.message
        : 'Internal server error'

    const errorMessage = Array.isArray(message) ? message[0] : message

    reply.status(status).send(fail(errorMessage))
  }
}
```

- [ ] **Step 6: Create apps/api/src/common/interceptors/response.interceptor.ts**

```typescript
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common'
import { Observable } from 'rxjs'
import { map } from 'rxjs/operators'
import { ApiResponse, ok } from '../types/api-response.type'

@Injectable()
export class ResponseInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  intercept(
    _context: ExecutionContext,
    next: CallHandler,
  ): Observable<ApiResponse<T>> {
    return next.handle().pipe(map((data) => ok(data)))
  }
}
```

- [ ] **Step 7: Create apps/api/src/app.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { ConfigModule } from '@nestjs/config'

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '../../.env',
    }),
  ],
})
export class AppModule {}
```

- [ ] **Step 8: Create apps/api/src/main.ts**

```typescript
import { NestFactory } from '@nestjs/core'
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify'
import { ValidationPipe } from '@nestjs/common'
import { AppModule } from './app.module'
import { AllExceptionsFilter } from './common/filters/http-exception.filter'
import { ResponseInterceptor } from './common/interceptors/response.interceptor'

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ logger: true }),
  )

  app.enableCors({ origin: process.env.WEB_URL ?? 'http://localhost:3000' })
  app.setGlobalPrefix('api/v1')
  app.useGlobalPipes(
    new ValidationPipe({ whitelist: true, transform: true }),
  )
  app.useGlobalFilters(new AllExceptionsFilter())
  app.useGlobalInterceptors(new ResponseInterceptor())

  const port = process.env.API_PORT ?? 3001
  await app.listen(port, '0.0.0.0')
  console.log(`API running on http://localhost:${port}`)
}

bootstrap()
```

- [ ] **Step 9: Install API dependencies**

```bash
cd apps/api && pnpm install
```

- [ ] **Step 10: Verify API starts**

```bash
cd apps/api && pnpm dev
```

Expected: `API running on http://localhost:3001`  
Hit `Ctrl+C` to stop.

- [ ] **Step 11: Commit**

```bash
cd ../..
git add apps/api
git commit -m "chore: scaffold NestJS API with Fastify, global filter and response interceptor"
```

---

## Task 3: Set up Prisma + PostgreSQL schema

**Files:**
- Create: `apps/api/prisma/schema.prisma`

Prerequisite: PostgreSQL running locally. If not, run:
```bash
docker run -d --name pos-postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=merxylab_pos -p 5432:5432 postgres:16
```

- [ ] **Step 1: Create apps/api/prisma/schema.prisma**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  SUPERADMIN
  ADMIN
  MANAGER
  CASHIER
  INVENTORY
}

model User {
  id           String          @id @default(cuid())
  name         String
  email        String          @unique
  passwordHash String          @map("password_hash")
  isActive     Boolean         @default(true) @map("is_active")
  createdAt    DateTime        @default(now()) @map("created_at")
  updatedAt    DateTime        @updatedAt @map("updated_at")
  storeRoles   UserStoreRole[]

  @@map("users")
}

model Store {
  id        String          @id @default(cuid())
  name      String
  address   String?
  phone     String?
  currency  String          @default("Ks")
  timezone  String          @default("Asia/Rangoon")
  isActive  Boolean         @default(true) @map("is_active")
  createdAt DateTime        @default(now()) @map("created_at")
  userRoles UserStoreRole[]

  @@map("stores")
}

model UserStoreRole {
  id      String @id @default(cuid())
  userId  String @map("user_id")
  storeId String @map("store_id")
  role    Role

  user  User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  store Store @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@unique([userId, storeId])
  @@map("user_store_roles")
}
```

- [ ] **Step 2: Run initial migration**

```bash
cd apps/api
pnpm prisma migrate dev --name init
```

Expected output: `✔ Generated Prisma Client`

- [ ] **Step 3: Seed a superadmin user for local dev**

Create `apps/api/prisma/seed.ts`:

```typescript
import { PrismaClient, Role } from '@prisma/client'
import * as bcrypt from 'bcryptjs'

const prisma = new PrismaClient()

async function main() {
  const hash = await bcrypt.hash('admin123', 10)

  const admin = await prisma.user.upsert({
    where: { email: 'admin@merxylab.com' },
    update: {},
    create: {
      name: 'Super Admin',
      email: 'admin@merxylab.com',
      passwordHash: hash,
    },
  })

  const storeA = await prisma.store.upsert({
    where: { id: 'store-a' },
    update: {},
    create: { id: 'store-a', name: 'Store A', address: 'Yangon' },
  })

  await prisma.userStoreRole.upsert({
    where: { userId_storeId: { userId: admin.id, storeId: storeA.id } },
    update: {},
    create: { userId: admin.id, storeId: storeA.id, role: Role.SUPERADMIN },
  })

  console.log('Seed complete. admin@merxylab.com / admin123')
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect())
```

Add to `apps/api/package.json` scripts:
```json
"db:seed": "ts-node prisma/seed.ts"
```

Add to `apps/api/package.json` top-level:
```json
"prisma": {
  "seed": "ts-node prisma/seed.ts"
}
```

- [ ] **Step 4: Run seed**

```bash
cd apps/api && pnpm db:seed
```

Expected: `Seed complete. admin@merxylab.com / admin123`

- [ ] **Step 5: Commit**

```bash
cd ../..
git add apps/api/prisma
git commit -m "chore: add Prisma schema with User, Store, UserStoreRole and seed"
```

---

## Task 4: Prisma service + Redis service in NestJS

**Files:**
- Create: `apps/api/src/prisma/prisma.module.ts`
- Create: `apps/api/src/prisma/prisma.service.ts`
- Create: `apps/api/src/redis/redis.module.ts`
- Create: `apps/api/src/redis/redis.service.ts`

Prerequisite: Redis running locally. If not:
```bash
docker run -d --name pos-redis -p 6379:6379 redis:7
```

- [ ] **Step 1: Create apps/api/src/prisma/prisma.service.ts**

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect()
  }
}
```

- [ ] **Step 2: Create apps/api/src/prisma/prisma.module.ts**

```typescript
import { Global, Module } from '@nestjs/common'
import { PrismaService } from './prisma.service'

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

- [ ] **Step 3: Create apps/api/src/redis/redis.service.ts**

```typescript
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'
import Redis from 'ioredis'

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis

  constructor(private config: ConfigService) {}

  onModuleInit() {
    this.client = new Redis(this.config.get<string>('REDIS_URL')!)
  }

  onModuleDestroy() {
    this.client.quit()
  }

  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.client.set(key, value, 'EX', ttlSeconds)
    } else {
      await this.client.set(key, value)
    }
  }

  async get(key: string): Promise<string | null> {
    return this.client.get(key)
  }

  async del(key: string): Promise<void> {
    await this.client.del(key)
  }

  async exists(key: string): Promise<boolean> {
    const count = await this.client.exists(key)
    return count > 0
  }
}
```

- [ ] **Step 4: Create apps/api/src/redis/redis.module.ts**

```typescript
import { Global, Module } from '@nestjs/common'
import { RedisService } from './redis.service'

@Global()
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}
```

- [ ] **Step 5: Register both modules in app.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { ConfigModule } from '@nestjs/config'
import { PrismaModule } from './prisma/prisma.module'
import { RedisModule } from './redis/redis.module'

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, envFilePath: '../../.env' }),
    PrismaModule,
    RedisModule,
  ],
})
export class AppModule {}
```

- [ ] **Step 6: Verify API starts without errors**

```bash
cd apps/api && pnpm dev
```

Expected: API starts, no connection errors.

- [ ] **Step 7: Commit**

```bash
cd ../..
git add apps/api/src/prisma apps/api/src/redis apps/api/src/app.module.ts
git commit -m "feat: add PrismaService and RedisService as global modules"
```

---

## Task 5: Shared packages — types and utils

**Files:**
- Create: `packages/types/package.json`
- Create: `packages/types/tsconfig.json`
- Create: `packages/types/src/enums.ts`
- Create: `packages/types/src/auth.types.ts`
- Create: `packages/types/src/api.types.ts`
- Create: `packages/types/src/index.ts`
- Create: `packages/utils/package.json`
- Create: `packages/utils/tsconfig.json`
- Create: `packages/utils/src/currency.ts`
- Create: `packages/utils/src/date.ts`
- Create: `packages/utils/src/index.ts`

- [ ] **Step 1: Create packages/types/package.json**

```json
{
  "name": "@merxy/types",
  "version": "0.0.1",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

- [ ] **Step 2: Create packages/types/tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "declaration": true,
    "outDir": "./dist",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Create packages/types/src/enums.ts**

```typescript
export enum Role {
  SUPERADMIN = 'SUPERADMIN',
  ADMIN = 'ADMIN',
  MANAGER = 'MANAGER',
  CASHIER = 'CASHIER',
  INVENTORY = 'INVENTORY',
}

export enum PaymentMethod {
  CASH = 'cash',
  CARD = 'card',
}

export enum OrderStatus {
  PENDING = 'pending',
  COMPLETED = 'completed',
  REFUNDED = 'refunded',
  HELD = 'held',
  CANCELLED = 'cancelled',
}

export enum StockMovementType {
  SALE = 'sale',
  RECEIVE = 'receive',
  ADJUST = 'adjust',
  TRANSFER_OUT = 'transfer_out',
  TRANSFER_IN = 'transfer_in',
}
```

- [ ] **Step 4: Create packages/types/src/auth.types.ts**

```typescript
import { Role } from './enums'

export interface JwtPayload {
  sub: string         // userId
  email: string
  storeId: string | null  // null for superadmin cross-store access
  role: Role
  deviceId: string    // used for refresh token Redis key
  iat?: number
  exp?: number
}

export interface AuthUser {
  id: string
  email: string
  name: string
  storeId: string | null
  role: Role
  deviceId: string
}

export interface StoreOption {
  id: string
  name: string
  role: Role
}

export interface LoginResponse {
  accessToken: string
  refreshToken: string
  user: {
    id: string
    name: string
    email: string
    role: Role
    storeId: string | null
  }
}

export interface StoreSelectionRequired {
  requiresStoreSelection: true
  stores: StoreOption[]
}
```

- [ ] **Step 5: Create packages/types/src/api.types.ts**

```typescript
export interface ApiMeta {
  total: number
  page: number
  limit: number
}

export interface ApiResponse<T = unknown> {
  success: boolean
  data: T | null
  error: string | null
  meta?: ApiMeta
}
```

- [ ] **Step 6: Create packages/types/src/index.ts**

```typescript
export * from './enums'
export * from './auth.types'
export * from './api.types'
```

- [ ] **Step 7: Create packages/utils/package.json**

```json
{
  "name": "@merxy/utils",
  "version": "0.0.1",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

- [ ] **Step 8: Create packages/utils/tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "declaration": true,
    "outDir": "./dist",
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

- [ ] **Step 9: Create packages/utils/src/currency.ts**

```typescript
/**
 * Format integer Kyats amount as "1,500 Ks"
 * All amounts are stored as integers (no decimals).
 */
export function formatKs(amount: number): string {
  return `${amount.toLocaleString('en-US')} Ks`
}

/**
 * Format for compact display: 4,821,000 → "4.82M Ks"
 */
export function formatKsCompact(amount: number): string {
  if (amount >= 1_000_000) {
    return `${(amount / 1_000_000).toFixed(2)}M Ks`
  }
  if (amount >= 1_000) {
    return `${(amount / 1_000).toFixed(1)}K Ks`
  }
  return formatKs(amount)
}
```

- [ ] **Step 10: Create packages/utils/src/date.ts**

```typescript
/**
 * Format date as "18 Apr 2026"
 */
export function formatDate(date: Date | string): string {
  return new Date(date).toLocaleDateString('en-GB', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  })
}

/**
 * Format datetime as "18 Apr 2026, 14:30"
 */
export function formatDateTime(date: Date | string): string {
  return new Date(date).toLocaleString('en-GB', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  })
}
```

- [ ] **Step 11: Create packages/utils/src/index.ts**

```typescript
export * from './currency'
export * from './date'
```

- [ ] **Step 12: Build packages**

```bash
cd packages/types && pnpm build
cd ../utils && pnpm build
```

Expected: `dist/` folders created in both packages.

- [ ] **Step 13: Commit**

```bash
cd ../..
git add packages/
git commit -m "feat: add shared @merxy/types and @merxy/utils packages"
```

---

## Task 6: Auth service (TDD)

**Files:**
- Create: `apps/api/src/auth/dto/login.dto.ts`
- Create: `apps/api/src/auth/dto/refresh.dto.ts`
- Create: `apps/api/src/auth/auth.service.ts`
- Create: `apps/api/src/auth/auth.service.spec.ts`

- [ ] **Step 1: Create apps/api/src/auth/dto/login.dto.ts**

```typescript
import { IsEmail, IsOptional, IsString, MinLength } from 'class-validator'

export class LoginDto {
  @IsEmail()
  email: string

  @IsString()
  @MinLength(6)
  password: string

  @IsOptional()
  @IsString()
  storeId?: string
}
```

- [ ] **Step 2: Create apps/api/src/auth/dto/refresh.dto.ts**

```typescript
import { IsString } from 'class-validator'

export class RefreshDto {
  @IsString()
  refreshToken: string
}
```

- [ ] **Step 3: Write failing auth service tests**

Create `apps/api/src/auth/auth.service.spec.ts`:

```typescript
import { Test, TestingModule } from '@nestjs/testing'
import { JwtService } from '@nestjs/jwt'
import { ConfigService } from '@nestjs/config'
import { UnauthorizedException, BadRequestException } from '@nestjs/common'
import { AuthService } from './auth.service'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'
import { Role } from '@prisma/client'
import * as bcrypt from 'bcryptjs'

const mockUser = {
  id: 'user-1',
  name: 'Test User',
  email: 'test@merxy.com',
  passwordHash: '',
  isActive: true,
  createdAt: new Date(),
  updatedAt: new Date(),
  storeRoles: [{ storeId: 'store-1', role: Role.CASHIER, store: { id: 'store-1', name: 'Store A' } }],
}

const mockPrisma = {
  user: {
    findUnique: jest.fn(),
  },
}

const mockRedis = {
  set: jest.fn(),
  get: jest.fn(),
  del: jest.fn(),
}

const mockJwt = {
  sign: jest.fn().mockReturnValue('mock-token'),
  verify: jest.fn(),
}

describe('AuthService', () => {
  let service: AuthService

  beforeEach(async () => {
    mockUser.passwordHash = await bcrypt.hash('password123', 10)

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: PrismaService, useValue: mockPrisma },
        { provide: RedisService, useValue: mockRedis },
        { provide: JwtService, useValue: mockJwt },
        { provide: ConfigService, useValue: { get: jest.fn().mockReturnValue('7d') } },
      ],
    }).compile()

    service = module.get<AuthService>(AuthService)
    jest.clearAllMocks()
  })

  describe('login', () => {
    it('throws UnauthorizedException when user not found', async () => {
      mockPrisma.user.findUnique.mockResolvedValue(null)
      await expect(service.login({ email: 'x@x.com', password: 'pass' }))
        .rejects.toThrow(UnauthorizedException)
    })

    it('throws UnauthorizedException when password is wrong', async () => {
      mockPrisma.user.findUnique.mockResolvedValue(mockUser)
      await expect(service.login({ email: mockUser.email, password: 'wrong' }))
        .rejects.toThrow(UnauthorizedException)
    })

    it('throws UnauthorizedException when user is inactive', async () => {
      mockPrisma.user.findUnique.mockResolvedValue({ ...mockUser, isActive: false })
      await expect(service.login({ email: mockUser.email, password: 'password123' }))
        .rejects.toThrow(UnauthorizedException)
    })

    it('returns requiresStoreSelection when user has multiple stores and no storeId given', async () => {
      const multiStoreUser = {
        ...mockUser,
        storeRoles: [
          { storeId: 'store-1', role: Role.CASHIER, store: { id: 'store-1', name: 'Store A' } },
          { storeId: 'store-2', role: Role.MANAGER, store: { id: 'store-2', name: 'Store B' } },
        ],
      }
      mockPrisma.user.findUnique.mockResolvedValue(multiStoreUser)
      const result = await service.login({ email: mockUser.email, password: 'password123' })
      expect(result).toHaveProperty('requiresStoreSelection', true)
      expect(result).toHaveProperty('stores')
    })

    it('returns tokens when credentials are valid and user has one store', async () => {
      mockPrisma.user.findUnique.mockResolvedValue(mockUser)
      mockJwt.sign.mockReturnValue('signed-token')
      const result = await service.login({ email: mockUser.email, password: 'password123' })
      expect(result).toHaveProperty('accessToken')
      expect(result).toHaveProperty('refreshToken')
      expect(mockRedis.set).toHaveBeenCalled()
    })

    it('uses provided storeId when user belongs to that store', async () => {
      mockPrisma.user.findUnique.mockResolvedValue(mockUser)
      mockJwt.sign.mockReturnValue('signed-token')
      const result = await service.login({ email: mockUser.email, password: 'password123', storeId: 'store-1' })
      expect(result).toHaveProperty('accessToken')
    })

    it('throws BadRequestException when provided storeId is not in user stores', async () => {
      mockPrisma.user.findUnique.mockResolvedValue(mockUser)
      await expect(service.login({ email: mockUser.email, password: 'password123', storeId: 'store-999' }))
        .rejects.toThrow(BadRequestException)
    })
  })

  describe('logout', () => {
    it('deletes the refresh token from Redis', async () => {
      await service.logout('user-1', 'device-1')
      expect(mockRedis.del).toHaveBeenCalledWith('refresh:user-1:device-1')
    })
  })
})
```

- [ ] **Step 4: Run tests — verify they FAIL**

```bash
cd apps/api && pnpm test auth.service
```

Expected: FAIL — `AuthService` not found / not implemented.

- [ ] **Step 5: Implement apps/api/src/auth/auth.service.ts**

```typescript
import {
  BadRequestException,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common'
import { JwtService } from '@nestjs/jwt'
import { ConfigService } from '@nestjs/config'
import { PrismaService } from '../prisma/prisma.service'
import { RedisService } from '../redis/redis.service'
import { LoginDto } from './dto/login.dto'
import { Role } from '@prisma/client'
import {
  AuthUser,
  JwtPayload,
  LoginResponse,
  StoreOption,
  StoreSelectionRequired,
} from '@merxy/types'
import * as bcrypt from 'bcryptjs'
import { randomUUID } from 'crypto'

const REFRESH_PREFIX = 'refresh'

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private redis: RedisService,
    private jwt: JwtService,
    private config: ConfigService,
  ) {}

  async login(
    dto: LoginDto,
  ): Promise<LoginResponse | StoreSelectionRequired> {
    const user = await this.prisma.user.findUnique({
      where: { email: dto.email },
      include: {
        storeRoles: { include: { store: { select: { id: true, name: true } } } },
      },
    })

    if (!user || !user.isActive) throw new UnauthorizedException('Invalid credentials')

    const passwordMatch = await bcrypt.compare(dto.password, user.passwordHash)
    if (!passwordMatch) throw new UnauthorizedException('Invalid credentials')

    const isSuperAdmin = user.storeRoles.some((r) => r.role === Role.SUPERADMIN)

    // Superadmin gets a store-less token
    if (isSuperAdmin) {
      return this.issueTokens(user.id, user.email, user.name, null, Role.SUPERADMIN)
    }

    // Single store — auto-select
    if (user.storeRoles.length === 1 && !dto.storeId) {
      const { storeId, role } = user.storeRoles[0]
      return this.issueTokens(user.id, user.email, user.name, storeId, role)
    }

    // Multiple stores — require selection
    if (user.storeRoles.length > 1 && !dto.storeId) {
      const stores: StoreOption[] = user.storeRoles.map((r) => ({
        id: r.storeId,
        name: r.store.name,
        role: r.role as unknown as import('@merxy/types').Role,
      }))
      return { requiresStoreSelection: true, stores }
    }

    // storeId provided — validate membership
    const membership = user.storeRoles.find((r) => r.storeId === dto.storeId)
    if (!membership) throw new BadRequestException('User does not belong to that store')

    return this.issueTokens(user.id, user.email, user.name, membership.storeId, membership.role)
  }

  async refreshToken(token: string): Promise<LoginResponse> {
    let payload: JwtPayload
    try {
      payload = this.jwt.verify<JwtPayload>(token, {
        secret: this.config.get('JWT_REFRESH_SECRET'),
      })
    } catch {
      throw new UnauthorizedException('Invalid refresh token')
    }

    const key = `${REFRESH_PREFIX}:${payload.sub}:${payload.deviceId}`
    const stored = await this.redis.get(key)
    if (!stored || stored !== token) {
      throw new UnauthorizedException('Refresh token has been revoked')
    }

    await this.redis.del(key)

    const user = await this.prisma.user.findUnique({
      where: { id: payload.sub },
      select: { id: true, email: true, name: true, isActive: true },
    })
    if (!user || !user.isActive) throw new UnauthorizedException('User inactive')

    return this.issueTokens(user.id, user.email, user.name, payload.storeId, payload.role, payload.deviceId)
  }

  async logout(userId: string, deviceId: string): Promise<void> {
    await this.redis.del(`${REFRESH_PREFIX}:${userId}:${deviceId}`)
  }

  private async issueTokens(
    userId: string,
    email: string,
    name: string,
    storeId: string | null,
    role: Role,
    existingDeviceId?: string,
  ): Promise<LoginResponse> {
    const deviceId = existingDeviceId ?? randomUUID()

    const payload: Omit<JwtPayload, 'iat' | 'exp'> = {
      sub: userId,
      email,
      storeId,
      role: role as unknown as import('@merxy/types').Role,
      deviceId,
    }

    const accessToken = this.jwt.sign(payload, {
      secret: this.config.get('JWT_ACCESS_SECRET'),
      expiresIn: this.config.get('JWT_ACCESS_EXPIRES_IN') ?? '15m',
    })

    const refreshToken = this.jwt.sign(payload, {
      secret: this.config.get('JWT_REFRESH_SECRET'),
      expiresIn: this.config.get('JWT_REFRESH_EXPIRES_IN') ?? '7d',
    })

    const ttlSeconds = 7 * 24 * 60 * 60 // 7 days
    await this.redis.set(`${REFRESH_PREFIX}:${userId}:${deviceId}`, refreshToken, ttlSeconds)

    return {
      accessToken,
      refreshToken,
      user: { id: userId, name, email, role: role as unknown as import('@merxy/types').Role, storeId },
    }
  }

  validateUser(payload: JwtPayload): AuthUser {
    return {
      id: payload.sub,
      email: payload.email,
      name: '',
      storeId: payload.storeId,
      role: payload.role,
      deviceId: payload.deviceId,
    }
  }
}
```

- [ ] **Step 6: Run tests — verify they PASS**

```bash
cd apps/api && pnpm test auth.service
```

Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
cd ../..
git add apps/api/src/auth
git commit -m "feat: implement AuthService with login, refresh, logout (TDD)"
```

---

## Task 7: JWT strategy + guards + decorators

**Files:**
- Create: `apps/api/src/auth/strategies/jwt.strategy.ts`
- Create: `apps/api/src/auth/guards/jwt-auth.guard.ts`
- Create: `apps/api/src/auth/guards/roles.guard.ts`
- Create: `apps/api/src/auth/decorators/roles.decorator.ts`
- Create: `apps/api/src/auth/decorators/current-user.decorator.ts`

- [ ] **Step 1: Create apps/api/src/auth/strategies/jwt.strategy.ts**

```typescript
import { Injectable } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { ExtractJwt, Strategy } from 'passport-jwt'
import { ConfigService } from '@nestjs/config'
import { AuthService } from '../auth.service'
import { JwtPayload } from '@merxy/types'

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService, private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.get<string>('JWT_ACCESS_SECRET')!,
    })
  }

  validate(payload: JwtPayload) {
    return this.authService.validateUser(payload)
  }
}
```

- [ ] **Step 2: Create apps/api/src/auth/guards/jwt-auth.guard.ts**

```typescript
import { Injectable } from '@nestjs/common'
import { AuthGuard } from '@nestjs/passport'

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

- [ ] **Step 3: Create apps/api/src/auth/decorators/roles.decorator.ts**

```typescript
import { SetMetadata } from '@nestjs/common'
import { Role } from '@merxy/types'

export const ROLES_KEY = 'roles'
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles)
```

- [ ] **Step 4: Create apps/api/src/auth/guards/roles.guard.ts**

```typescript
import {
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  Injectable,
} from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { Role } from '@merxy/types'
import { ROLES_KEY } from '../decorators/roles.decorator'
import { AuthUser } from '@merxy/types'

const ROLE_HIERARCHY: Record<Role, number> = {
  [Role.SUPERADMIN]: 5,
  [Role.ADMIN]: 4,
  [Role.MANAGER]: 3,
  [Role.CASHIER]: 2,
  [Role.INVENTORY]: 1,
}

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ])

    if (!requiredRoles || requiredRoles.length === 0) return true

    const request = context.switchToHttp().getRequest()
    const user: AuthUser = request.user

    if (!user) throw new ForbiddenException('Not authenticated')

    const userLevel = ROLE_HIERARCHY[user.role] ?? 0
    const minRequired = Math.min(...requiredRoles.map((r) => ROLE_HIERARCHY[r]))

    if (userLevel < minRequired) {
      throw new ForbiddenException('Insufficient permissions')
    }

    return true
  }
}
```

- [ ] **Step 5: Create apps/api/src/auth/decorators/current-user.decorator.ts**

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common'
import { AuthUser } from '@merxy/types'

export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): AuthUser => {
    return ctx.switchToHttp().getRequest().user
  },
)
```

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/auth/strategies apps/api/src/auth/guards apps/api/src/auth/decorators
git commit -m "feat: add JWT strategy, RolesGuard, and auth decorators"
```

---

## Task 8: Auth controller + module (TDD)

**Files:**
- Create: `apps/api/src/auth/auth.controller.ts`
- Create: `apps/api/src/auth/auth.controller.spec.ts`
- Create: `apps/api/src/auth/auth.module.ts`
- Modify: `apps/api/src/app.module.ts`

- [ ] **Step 1: Write failing controller tests**

Create `apps/api/src/auth/auth.controller.spec.ts`:

```typescript
import { Test, TestingModule } from '@nestjs/testing'
import { AuthController } from './auth.controller'
import { AuthService } from './auth.service'

const mockAuthService = {
  login: jest.fn(),
  refreshToken: jest.fn(),
  logout: jest.fn(),
}

describe('AuthController', () => {
  let controller: AuthController

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [AuthController],
      providers: [{ provide: AuthService, useValue: mockAuthService }],
    }).compile()

    controller = module.get<AuthController>(AuthController)
    jest.clearAllMocks()
  })

  it('POST /login calls authService.login with dto', async () => {
    const dto = { email: 'a@b.com', password: 'pass123' }
    mockAuthService.login.mockResolvedValue({ accessToken: 'tok' })
    const result = await controller.login(dto as any)
    expect(mockAuthService.login).toHaveBeenCalledWith(dto)
    expect(result).toEqual({ accessToken: 'tok' })
  })

  it('POST /refresh calls authService.refreshToken with token', async () => {
    mockAuthService.refreshToken.mockResolvedValue({ accessToken: 'new-tok' })
    const result = await controller.refresh({ refreshToken: 'old-tok' })
    expect(mockAuthService.refreshToken).toHaveBeenCalledWith('old-tok')
    expect(result).toEqual({ accessToken: 'new-tok' })
  })

  it('DELETE /logout calls authService.logout with user ids', async () => {
    const user = { id: 'u1', deviceId: 'd1' }
    await controller.logout(user as any)
    expect(mockAuthService.logout).toHaveBeenCalledWith('u1', 'd1')
  })
})
```

- [ ] **Step 2: Run tests — verify they FAIL**

```bash
cd apps/api && pnpm test auth.controller
```

Expected: FAIL — `AuthController` not found.

- [ ] **Step 3: Create apps/api/src/auth/auth.controller.ts**

```typescript
import { Body, Controller, Delete, Post, UseGuards } from '@nestjs/common'
import { AuthService } from './auth.service'
import { LoginDto } from './dto/login.dto'
import { RefreshDto } from './dto/refresh.dto'
import { JwtAuthGuard } from './guards/jwt-auth.guard'
import { CurrentUser } from './decorators/current-user.decorator'
import { AuthUser } from '@merxy/types'

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto)
  }

  @Post('refresh')
  refresh(@Body() dto: RefreshDto) {
    return this.authService.refreshToken(dto.refreshToken)
  }

  @Delete('logout')
  @UseGuards(JwtAuthGuard)
  logout(@CurrentUser() user: AuthUser) {
    return this.authService.logout(user.id, user.deviceId)
  }
}
```

- [ ] **Step 4: Run tests — verify they PASS**

```bash
cd apps/api && pnpm test auth.controller
```

Expected: All tests pass.

- [ ] **Step 5: Create apps/api/src/auth/auth.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { JwtModule } from '@nestjs/jwt'
import { PassportModule } from '@nestjs/passport'
import { AuthController } from './auth.controller'
import { AuthService } from './auth.service'
import { JwtStrategy } from './strategies/jwt.strategy'

@Module({
  imports: [
    PassportModule,
    JwtModule.register({}),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

- [ ] **Step 6: Register AuthModule in app.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { ConfigModule } from '@nestjs/config'
import { PrismaModule } from './prisma/prisma.module'
import { RedisModule } from './redis/redis.module'
import { AuthModule } from './auth/auth.module'

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, envFilePath: '../../.env' }),
    PrismaModule,
    RedisModule,
    AuthModule,
  ],
})
export class AppModule {}
```

- [ ] **Step 7: Add @merxy/types to API dependencies**

In `apps/api/package.json` add to `dependencies`:
```json
"@merxy/types": "workspace:*",
"@merxy/utils": "workspace:*"
```

Then install:
```bash
cd apps/api && pnpm install
```

- [ ] **Step 8: Start API and test login manually**

```bash
cd apps/api && pnpm dev
```

In a second terminal:
```bash
curl -s -X POST http://localhost:3001/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@merxylab.com","password":"admin123"}' | jq .
```

Expected:
```json
{
  "success": true,
  "data": {
    "accessToken": "...",
    "refreshToken": "...",
    "user": { "id": "...", "name": "Super Admin", "email": "admin@merxylab.com", "role": "SUPERADMIN", "storeId": null }
  },
  "error": null
}
```

- [ ] **Step 9: Commit**

```bash
cd ../..
git add apps/api/src/auth/auth.controller.ts apps/api/src/auth/auth.controller.spec.ts apps/api/src/auth/auth.module.ts apps/api/src/app.module.ts
git commit -m "feat: wire AuthModule with controller, JWT strategy, and guards"
```

---

## Task 9: Scaffold Next.js web app

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/tsconfig.json`
- Create: `apps/web/next.config.ts`
- Create: `apps/web/postcss.config.js`
- Create: `apps/web/components.json`

- [ ] **Step 1: Create apps/web/package.json**

```json
{
  "name": "web",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "@merxy/types": "workspace:*",
    "@merxy/utils": "workspace:*",
    "@tanstack/react-query": "^5.40.0",
    "axios": "^1.7.2",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.1",
    "jose": "^5.4.0",
    "lucide-react": "^0.395.0",
    "next": "^14.2.4",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "tailwind-merge": "^2.3.0",
    "zustand": "^4.5.4"
  },
  "devDependencies": {
    "@types/node": "^20.14.0",
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38",
    "tailwindcss": "^3.4.4",
    "typescript": "^5.4.0"
  }
}
```

- [ ] **Step 2: Create apps/web/tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: Create apps/web/next.config.ts**

```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  transpilePackages: ['@merxy/types', '@merxy/utils'],
}

export default config
```

- [ ] **Step 4: Create apps/web/postcss.config.js**

```js
module.exports = {
  plugins: { tailwindcss: {}, autoprefixer: {} },
}
```

- [ ] **Step 5: Create apps/web/components.json (shadcn/ui config)**

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "src/app/globals.css",
    "baseColor": "slate",
    "cssVariables": false
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

- [ ] **Step 6: Install web dependencies**

```bash
cd apps/web && pnpm install
```

- [ ] **Step 7: Commit**

```bash
cd ../..
git add apps/web
git commit -m "chore: scaffold Next.js 14 web app with shadcn/ui config"
```

---

## Task 10: Design tokens + Tailwind config

**Files:**
- Create: `apps/web/tailwind.config.ts`
- Create: `apps/web/src/app/globals.css`
- Create: `apps/web/src/lib/utils.ts`

- [ ] **Step 1: Create apps/web/tailwind.config.ts**

```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        // Primary — green, used only for CTAs and positive states
        primary: {
          DEFAULT: '#059669',
          light: '#34d399',
          tint: '#d1fae5',
        },
        // Canvas / backgrounds
        canvas: '#edf0f4',
        surface: '#f7f9fb',
        // Sidebar
        sidebar: {
          bg: '#1a202c',
          hover: '#2d3748',
          active: 'rgba(5,150,105,0.15)',
          text: '#a0aec0',
          'text-active': '#34d399',
        },
        // Text
        'text-dark': '#1a202c',
        'text-muted': '#718096',
        // Borders
        border: '#d1d8e0',
        // Semantic
        danger: '#e53e3e',
        warning: '#d69e2e',
        info: '#3182ce',
      },
      fontSize: {
        // Minimum 16px for inputs (prevents iOS zoom)
        'input': ['16px', { lineHeight: '1.5' }],
      },
      minHeight: {
        touch: '44px',   // minimum tap target
      },
      minWidth: {
        touch: '44px',
      },
      screens: {
        mobile: '375px',
        tablet: '768px',
        desktop: '1280px',
      },
    },
  },
  plugins: [],
}

export default config
```

- [ ] **Step 2: Create apps/web/src/app/globals.css**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  * {
    box-sizing: border-box;
  }

  html {
    -webkit-tap-highlight-color: transparent;
  }

  body {
    @apply bg-canvas text-text-dark;
    font-family: -apple-system, BlinkMacSystemFont, 'Inter', 'Segoe UI', sans-serif;
  }

  /* Prevent iOS font size scaling on inputs */
  input, select, textarea {
    font-size: 16px;
  }
}

@layer components {
  /* Shared nav item */
  .nav-item {
    @apply flex items-center gap-2.5 px-2.5 py-[7px] rounded-md text-sidebar-text text-xs font-medium cursor-pointer transition-colors duration-100;
    min-height: 44px;
  }
  .nav-item:hover {
    @apply bg-sidebar-hover text-[#e2e8f0];
  }
  .nav-item.active {
    @apply bg-sidebar-active text-sidebar-text-active;
  }

  /* KPI card */
  .kpi-card {
    @apply bg-surface rounded-xl p-4 border border-border;
  }

  /* Table styles */
  .data-table th {
    @apply text-left px-3 py-2 text-text-muted text-[10px] uppercase tracking-wider border-b border-border bg-canvas font-semibold;
  }
  .data-table td {
    @apply px-3 py-2 border-b border-[#e8ecf0] text-sm text-[#374151];
  }
  .data-table tr:last-child td {
    @apply border-b-0;
  }

  /* Badge */
  .badge {
    @apply inline-block px-2 py-0.5 rounded-full text-[10px] font-bold;
  }
  .badge-green { @apply bg-primary-tint text-[#065f46]; }
  .badge-yellow { @apply bg-[#fef3c7] text-[#92400e]; }
  .badge-red { @apply bg-[#fee2e2] text-[#991b1b]; }
  .badge-blue { @apply bg-[#dbeafe] text-[#1e40af]; }
  .badge-gray { @apply bg-[#e2e8f0] text-[#4a5568]; }

  /* Primary button */
  .btn-primary {
    @apply bg-primary text-white font-semibold rounded-lg px-4 py-2 text-sm cursor-pointer border border-primary transition-opacity hover:opacity-90 min-h-touch;
  }

  /* Ghost button */
  .btn-ghost {
    @apply bg-surface text-[#374151] font-semibold rounded-lg px-4 py-2 text-sm cursor-pointer border border-border transition-colors hover:bg-canvas min-h-touch;
  }
}
```

- [ ] **Step 3: Create apps/web/src/lib/utils.ts**

```typescript
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/tailwind.config.ts apps/web/src/app/globals.css apps/web/src/lib/utils.ts
git commit -m "feat: add design tokens to Tailwind config and global CSS"
```

---

## Task 11: API client + Zustand auth store

**Files:**
- Create: `apps/web/src/lib/api.ts`
- Create: `apps/web/src/lib/auth.ts`
- Create: `apps/web/src/store/auth.store.ts`

- [ ] **Step 1: Create apps/web/src/lib/api.ts**

```typescript
import axios from 'axios'

export const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001/api/v1',
  headers: { 'Content-Type': 'application/json' },
})

// Attach access token from localStorage on every request
api.interceptors.request.use((config) => {
  if (typeof window !== 'undefined') {
    const token = localStorage.getItem('accessToken')
    if (token) config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Refresh token on 401
api.interceptors.response.use(
  (res) => res,
  async (error) => {
    const original = error.config
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true
      const refreshToken = localStorage.getItem('refreshToken')
      if (!refreshToken) {
        localStorage.clear()
        window.location.href = '/login'
        return Promise.reject(error)
      }
      try {
        const { data } = await axios.post(
          `${process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001/api/v1'}/auth/refresh`,
          { refreshToken },
        )
        const { accessToken, refreshToken: newRefresh } = data.data
        localStorage.setItem('accessToken', accessToken)
        localStorage.setItem('refreshToken', newRefresh)
        original.headers.Authorization = `Bearer ${accessToken}`
        return api(original)
      } catch {
        localStorage.clear()
        window.location.href = '/login'
      }
    }
    return Promise.reject(error)
  },
)
```

- [ ] **Step 2: Create apps/web/src/lib/auth.ts**

```typescript
import { decodeJwt } from 'jose'
import { JwtPayload, Role } from '@merxy/types'

export function decodeAccessToken(token: string): JwtPayload | null {
  try {
    return decodeJwt(token) as unknown as JwtPayload
  } catch {
    return null
  }
}

export function getStoredAuth(): { accessToken: string; payload: JwtPayload } | null {
  if (typeof window === 'undefined') return null
  const token = localStorage.getItem('accessToken')
  if (!token) return null
  const payload = decodeAccessToken(token)
  if (!payload) return null
  // Check not expired (exp is in seconds)
  if (payload.exp && payload.exp * 1000 < Date.now()) return null
  return { accessToken: token, payload }
}

export const ROLE_HIERARCHY: Record<Role, number> = {
  [Role.SUPERADMIN]: 5,
  [Role.ADMIN]: 4,
  [Role.MANAGER]: 3,
  [Role.CASHIER]: 2,
  [Role.INVENTORY]: 1,
}

export function hasMinRole(userRole: Role, minRole: Role): boolean {
  return ROLE_HIERARCHY[userRole] >= ROLE_HIERARCHY[minRole]
}
```

- [ ] **Step 3: Create apps/web/src/store/auth.store.ts**

```typescript
import { create } from 'zustand'
import { Role, StoreOption, LoginResponse } from '@merxy/types'
import { api } from '@/lib/api'

interface AuthState {
  userId: string | null
  userName: string | null
  email: string | null
  role: Role | null
  storeId: string | null
  isAuthenticated: boolean
  pendingStores: StoreOption[] | null

  login: (email: string, password: string, storeId?: string) => Promise<void>
  selectStore: (email: string, password: string, storeId: string) => Promise<void>
  logout: () => Promise<void>
  hydrate: () => void
}

export const useAuthStore = create<AuthState>((set, get) => ({
  userId: null,
  userName: null,
  email: null,
  role: null,
  storeId: null,
  isAuthenticated: false,
  pendingStores: null,

  hydrate() {
    const token = localStorage.getItem('accessToken')
    const userRaw = localStorage.getItem('authUser')
    if (token && userRaw) {
      try {
        const user = JSON.parse(userRaw)
        set({ ...user, isAuthenticated: true })
      } catch {
        localStorage.clear()
      }
    }
  },

  async login(email, password, storeId) {
    const { data } = await api.post<{ data: LoginResponse | { requiresStoreSelection: true; stores: StoreOption[] } }>(
      '/auth/login',
      { email, password, storeId },
    )
    const body = data.data

    if ('requiresStoreSelection' in body) {
      set({ pendingStores: body.stores })
      return
    }

    const res = body as LoginResponse
    localStorage.setItem('accessToken', res.accessToken)
    localStorage.setItem('refreshToken', res.refreshToken)
    const userState = {
      userId: res.user.id,
      userName: res.user.name,
      email: res.user.email,
      role: res.user.role,
      storeId: res.user.storeId,
    }
    localStorage.setItem('authUser', JSON.stringify(userState))
    set({ ...userState, isAuthenticated: true, pendingStores: null })
  },

  async selectStore(email, password, storeId) {
    return get().login(email, password, storeId)
  },

  async logout() {
    try {
      await api.delete('/auth/logout')
    } catch { /* ignore */ }
    localStorage.clear()
    set({
      userId: null, userName: null, email: null,
      role: null, storeId: null, isAuthenticated: false, pendingStores: null,
    })
  },
}))
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/lib apps/web/src/store
git commit -m "feat: add axios API client with token refresh and Zustand auth store"
```

---

## Task 12: Login page

**Files:**
- Create: `apps/web/src/app/(auth)/login/page.tsx`
- Create: `apps/web/src/app/layout.tsx`

- [ ] **Step 1: Create apps/web/src/app/layout.tsx**

```tsx
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: 'MerxyLab POS',
  description: 'Multi-store Point of Sale',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

- [ ] **Step 2: Create apps/web/src/app/(auth)/login/page.tsx**

```tsx
'use client'

import { useState, FormEvent } from 'react'
import { useRouter } from 'next/navigation'
import { useAuthStore } from '@/store/auth.store'
import { Role, StoreOption } from '@merxy/types'

export default function LoginPage() {
  const router = useRouter()
  const { login, selectStore, pendingStores } = useAuthStore()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [loading, setLoading] = useState(false)

  // Redirect destination by role
  function redirectByRole(role: Role) {
    const routes: Record<Role, string> = {
      [Role.SUPERADMIN]: '/dashboard',
      [Role.ADMIN]: '/admin',
      [Role.MANAGER]: '/dashboard',
      [Role.CASHIER]: '/pos',
      [Role.INVENTORY]: '/stock',
    }
    router.push(routes[role] ?? '/pos')
  }

  async function handleSubmit(e: FormEvent) {
    e.preventDefault()
    setError('')
    setLoading(true)
    try {
      await login(email, password)
      const { role, pendingStores } = useAuthStore.getState()
      if (!pendingStores && role) redirectByRole(role)
    } catch (err: any) {
      setError(err?.response?.data?.error ?? 'Login failed')
    } finally {
      setLoading(false)
    }
  }

  async function handleStoreSelect(store: StoreOption) {
    setLoading(true)
    try {
      await selectStore(email, password, store.id)
      const { role } = useAuthStore.getState()
      if (role) redirectByRole(role)
    } catch (err: any) {
      setError(err?.response?.data?.error ?? 'Failed to select store')
    } finally {
      setLoading(false)
    }
  }

  if (pendingStores) {
    return (
      <div className="min-h-screen bg-canvas flex items-center justify-center p-4">
        <div className="bg-surface border border-border rounded-xl p-8 w-full max-w-sm shadow-sm">
          <div className="flex justify-center mb-6">
            <div className="w-10 h-10 bg-primary rounded-xl flex items-center justify-center text-white font-bold text-lg">M</div>
          </div>
          <h1 className="text-text-dark font-bold text-xl text-center mb-1">Select Store</h1>
          <p className="text-text-muted text-sm text-center mb-6">You have access to multiple stores</p>
          <div className="flex flex-col gap-3">
            {pendingStores.map((store) => (
              <button
                key={store.id}
                onClick={() => handleStoreSelect(store)}
                disabled={loading}
                className="w-full p-4 bg-canvas border border-border rounded-lg text-left hover:border-primary hover:bg-primary-tint transition-colors min-h-touch"
              >
                <div className="font-semibold text-text-dark text-sm">{store.name}</div>
                <div className="text-text-muted text-xs mt-0.5 capitalize">{store.role.toLowerCase()}</div>
              </button>
            ))}
          </div>
          {error && <p className="text-danger text-xs mt-3 text-center">{error}</p>}
        </div>
      </div>
    )
  }

  return (
    <div className="min-h-screen bg-canvas flex items-center justify-center p-4">
      <div className="bg-surface border border-border rounded-xl p-8 w-full max-w-sm shadow-sm">
        <div className="flex justify-center mb-6">
          <div className="w-10 h-10 bg-primary rounded-xl flex items-center justify-center text-white font-bold text-lg">M</div>
        </div>
        <h1 className="text-text-dark font-bold text-xl text-center mb-1">MerxyLab POS</h1>
        <p className="text-text-muted text-sm text-center mb-6">Sign in to your account</p>

        <form onSubmit={handleSubmit} className="flex flex-col gap-4">
          <div className="flex flex-col gap-1.5">
            <label className="text-[11px] font-bold text-[#4a5568] uppercase tracking-wide">Email</label>
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              placeholder="you@merxylab.com"
              required
              className="w-full px-3 py-2.5 border border-border rounded-lg text-text-dark bg-white focus:outline-none focus:border-primary focus:ring-2 focus:ring-primary/10 min-h-touch"
            />
          </div>
          <div className="flex flex-col gap-1.5">
            <label className="text-[11px] font-bold text-[#4a5568] uppercase tracking-wide">Password</label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              placeholder="••••••••"
              required
              className="w-full px-3 py-2.5 border border-border rounded-lg text-text-dark bg-white focus:outline-none focus:border-primary focus:ring-2 focus:ring-primary/10 min-h-touch"
            />
          </div>
          {error && <p className="text-danger text-xs">{error}</p>}
          <button
            type="submit"
            disabled={loading}
            className="w-full py-3 bg-primary text-white font-bold rounded-lg text-sm hover:opacity-90 transition-opacity disabled:opacity-60 min-h-touch mt-1"
          >
            {loading ? 'Signing in…' : 'Sign In'}
          </button>
        </form>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/app
git commit -m "feat: add login page with store selection flow"
```

---

## Task 13: Shared layout shell + panel layouts

**Files:**
- Create: `apps/web/src/components/layout/AppShell.tsx`
- Create: `apps/web/src/components/layout/Sidebar.tsx`
- Create: `apps/web/src/components/layout/TopBar.tsx`
- Create: `apps/web/src/app/(dashboard)/layout.tsx`
- Create: `apps/web/src/app/(stock)/layout.tsx`
- Create: `apps/web/src/app/(admin)/layout.tsx`
- Create: `apps/web/src/app/(pos)/layout.tsx`

- [ ] **Step 1: Create apps/web/src/components/layout/Sidebar.tsx**

```tsx
'use client'

import { usePathname, useRouter } from 'next/navigation'
import { useAuthStore } from '@/store/auth.store'
import { cn } from '@/lib/utils'

interface NavItem {
  label: string
  href: string
  icon?: string
}

interface NavSection {
  label: string
  items: NavItem[]
}

interface SidebarProps {
  panelLabel: string
  sections: NavSection[]
}

export function Sidebar({ panelLabel, sections }: SidebarProps) {
  const pathname = usePathname()
  const router = useRouter()
  const { userName, role, storeId, logout } = useAuthStore()

  async function handleLogout() {
    await logout()
    router.push('/login')
  }

  const initials = userName
    ? userName.split(' ').map((n) => n[0]).join('').toUpperCase().slice(0, 2)
    : '??'

  return (
    <aside className="w-[216px] bg-sidebar-bg flex flex-col flex-shrink-0 h-full">
      {/* Logo */}
      <div className="px-3.5 pt-4 pb-3.5 border-b border-[#2d3748] flex items-center gap-2.5">
        <div className="w-7 h-7 bg-primary rounded-[7px] flex items-center justify-center text-white font-bold text-sm flex-shrink-0">
          M
        </div>
        <div>
          <div className="text-[#f1f5f9] font-bold text-[13px] leading-tight">MerxyLab</div>
          <div className="text-[#718096] text-[10px]">{panelLabel}</div>
        </div>
      </div>

      {/* Nav */}
      <nav className="flex-1 px-1.5 py-2.5 overflow-y-auto">
        {sections.map((section) => (
          <div key={section.label}>
            <div className="px-2 pt-2.5 pb-1 text-[9px] font-bold uppercase tracking-widest text-[#4a5568]">
              {section.label}
            </div>
            {section.items.map((item) => {
              const isActive = pathname === item.href || pathname.startsWith(item.href + '/')
              return (
                <button
                  key={item.href}
                  onClick={() => router.push(item.href)}
                  className={cn(
                    'nav-item w-full text-left mb-0.5',
                    isActive && 'active',
                  )}
                >
                  <div className="w-3.5 h-3.5 bg-current rounded-sm opacity-80 flex-shrink-0" />
                  {item.label}
                </button>
              )
            })}
          </div>
        ))}
      </nav>

      {/* User footer */}
      <div className="px-1.5 py-2.5 border-t border-[#2d3748]">
        <div className="flex items-center gap-2 px-2 py-1.5 rounded-md">
          <div className="w-[26px] h-[26px] bg-[#2d3748] rounded-full flex items-center justify-center text-[10px] font-bold text-[#a0aec0] flex-shrink-0">
            {initials}
          </div>
          <div className="flex-1 min-w-0">
            <div className="text-[#e2e8f0] font-semibold text-[11px] truncate">{userName}</div>
            <div className="text-[9px] bg-[#2d3748] text-[#718096] px-1.5 rounded-full inline-block mt-0.5">
              {role?.toLowerCase()} {storeId ? `· ${storeId.slice(0, 8)}` : ''}
            </div>
          </div>
        </div>
        <button
          onClick={handleLogout}
          className="w-full mt-1 px-2 py-1.5 text-[11px] text-[#718096] hover:text-[#e2e8f0] hover:bg-[#2d3748] rounded-md transition-colors text-left min-h-touch"
        >
          Sign out
        </button>
      </div>
    </aside>
  )
}
```

- [ ] **Step 2: Create apps/web/src/components/layout/TopBar.tsx**

```tsx
interface TopBarProps {
  title: string
  actions?: React.ReactNode
}

export function TopBar({ title, actions }: TopBarProps) {
  return (
    <header className="bg-surface border-b border-border h-[50px] px-5 flex items-center justify-between flex-shrink-0">
      <h1 className="text-[15px] font-bold text-text-dark">{title}</h1>
      {actions && <div className="flex gap-2 items-center">{actions}</div>}
    </header>
  )
}
```

- [ ] **Step 3: Create apps/web/src/components/layout/AppShell.tsx**

```tsx
import { Sidebar } from './Sidebar'

interface NavItem { label: string; href: string }
interface NavSection { label: string; items: NavItem[] }

interface AppShellProps {
  panelLabel: string
  sections: NavSection[]
  children: React.ReactNode
}

export function AppShell({ panelLabel, sections, children }: AppShellProps) {
  return (
    <div className="flex h-screen overflow-hidden bg-canvas">
      <Sidebar panelLabel={panelLabel} sections={sections} />
      <main className="flex-1 flex flex-col overflow-hidden bg-canvas">
        {children}
      </main>
    </div>
  )
}
```

- [ ] **Step 4: Create apps/web/src/app/(dashboard)/layout.tsx**

```tsx
import { AppShell } from '@/components/layout/AppShell'

const SECTIONS = [
  {
    label: 'Overview',
    items: [
      { label: 'Overview', href: '/dashboard' },
      { label: 'Sales Analytics', href: '/dashboard/sales' },
      { label: 'Profit Report', href: '/dashboard/profit' },
    ],
  },
  {
    label: 'Insights',
    items: [
      { label: 'Product Performance', href: '/dashboard/products' },
      { label: 'Staff Activity', href: '/dashboard/staff' },
      { label: 'Refund Report', href: '/dashboard/refunds' },
      { label: 'Payment Summary', href: '/dashboard/payments' },
    ],
  },
  {
    label: 'Export',
    items: [{ label: 'Export Reports', href: '/dashboard/export' }],
  },
]

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <AppShell panelLabel="Dashboard" sections={SECTIONS}>
      {children}
    </AppShell>
  )
}
```

- [ ] **Step 5: Create apps/web/src/app/(stock)/layout.tsx**

```tsx
import { AppShell } from '@/components/layout/AppShell'

const SECTIONS = [
  {
    label: 'Inventory',
    items: [
      { label: 'Products', href: '/stock/products' },
      { label: 'Categories', href: '/stock/categories' },
      { label: 'Stock Overview', href: '/stock/overview' },
    ],
  },
  {
    label: 'Movements',
    items: [
      { label: 'Adjustments', href: '/stock/adjustments' },
      { label: 'Transfers', href: '/stock/transfers' },
      { label: 'Stock Count', href: '/stock/count' },
    ],
  },
  {
    label: 'Purchasing',
    items: [
      { label: 'Purchase Orders', href: '/stock/purchase-orders' },
      { label: 'Suppliers', href: '/stock/suppliers' },
      { label: 'Low Stock', href: '/stock/low-stock' },
    ],
  },
]

export default function StockLayout({ children }: { children: React.ReactNode }) {
  return (
    <AppShell panelLabel="Stock" sections={SECTIONS}>
      {children}
    </AppShell>
  )
}
```

- [ ] **Step 6: Create apps/web/src/app/(admin)/layout.tsx**

```tsx
import { AppShell } from '@/components/layout/AppShell'

const SECTIONS = [
  {
    label: 'Users & Access',
    items: [
      { label: 'User Management', href: '/admin/users' },
      { label: 'Roles & Permissions', href: '/admin/roles' },
    ],
  },
  {
    label: 'Store Config',
    items: [
      { label: 'Store Settings', href: '/admin/stores' },
      { label: 'Tax Settings', href: '/admin/tax' },
      { label: 'Receipt Settings', href: '/admin/receipts' },
      { label: 'Payment Settings', href: '/admin/payments' },
    ],
  },
  {
    label: 'System',
    items: [
      { label: 'Devices / Sessions', href: '/admin/devices' },
      { label: 'Audit Logs', href: '/admin/audit' },
      { label: 'System Logs', href: '/admin/logs' },
    ],
  },
]

export default function AdminLayout({ children }: { children: React.ReactNode }) {
  return (
    <AppShell panelLabel="Admin" sections={SECTIONS}>
      {children}
    </AppShell>
  )
}
```

- [ ] **Step 7: Create apps/web/src/app/(pos)/layout.tsx**

```tsx
// POS is full-screen — no sidebar shell
export default function PosLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="h-screen overflow-hidden bg-canvas flex flex-col">
      {children}
    </div>
  )
}
```

- [ ] **Step 8: Commit**

```bash
git add apps/web/src/components apps/web/src/app
git commit -m "feat: add shared AppShell, Sidebar, TopBar and four panel layouts"
```

---

## Task 14: Next.js middleware for route protection

**Files:**
- Create: `apps/web/middleware.ts`
- Create: `apps/web/src/app/(auth)/login/page.tsx` (already done in Task 12)

- [ ] **Step 1: Create apps/web/middleware.ts**

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { jwtVerify } from 'jose'
import { Role } from '@merxy/types'

const ACCESS_SECRET = new TextEncoder().encode(
  process.env.JWT_ACCESS_SECRET ?? 'change-me-access-secret-min-32-chars',
)

// Minimum role required per path prefix
const ROUTE_ROLES: { prefix: string; minRole: Role }[] = [
  { prefix: '/pos', minRole: Role.CASHIER },
  { prefix: '/stock', minRole: Role.INVENTORY },
  { prefix: '/dashboard', minRole: Role.MANAGER },
  { prefix: '/admin', minRole: Role.ADMIN },
]

const ROLE_LEVEL: Record<Role, number> = {
  [Role.SUPERADMIN]: 5,
  [Role.ADMIN]: 4,
  [Role.MANAGER]: 3,
  [Role.CASHIER]: 2,
  [Role.INVENTORY]: 1,
}

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // Public routes
  if (pathname.startsWith('/login') || pathname.startsWith('/_next') || pathname.startsWith('/api')) {
    return NextResponse.next()
  }

  const route = ROUTE_ROLES.find((r) => pathname.startsWith(r.prefix))
  if (!route) return NextResponse.next()

  const token = request.cookies.get('accessToken')?.value
  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  try {
    const { payload } = await jwtVerify(token, ACCESS_SECRET)
    const userRole = payload.role as Role
    if (ROLE_LEVEL[userRole] < ROLE_LEVEL[route.minRole]) {
      return NextResponse.redirect(new URL('/login', request.url))
    }
    return NextResponse.next()
  } catch {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}

export const config = {
  matcher: ['/pos/:path*', '/dashboard/:path*', '/stock/:path*', '/admin/:path*'],
}
```

Note: The middleware reads the token from a cookie. Update the auth store's `login` method to also set a cookie (httpOnly cookies require server action — for now use `document.cookie`). Add to `auth.store.ts` after setting localStorage:

```typescript
// Also set cookie for middleware (non-httpOnly for client-side SSR middleware)
document.cookie = `accessToken=${res.accessToken}; path=/; max-age=${15 * 60}; SameSite=Strict`
```

And in `logout`:
```typescript
document.cookie = 'accessToken=; path=/; max-age=0'
```

- [ ] **Step 2: Add NEXT_PUBLIC_API_URL and JWT_ACCESS_SECRET to web .env**

Create `apps/web/.env.local`:
```env
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
JWT_ACCESS_SECRET=change-me-access-secret-min-32-chars
```

- [ ] **Step 3: Add stub pages for each panel so we can test routing**

Create `apps/web/src/app/(dashboard)/dashboard/page.tsx`:
```tsx
export default function DashboardPage() {
  return <div className="p-6"><h1 className="text-xl font-bold">Dashboard</h1></div>
}
```

Create `apps/web/src/app/(pos)/pos/page.tsx`:
```tsx
export default function PosPage() {
  return <div className="p-6"><h1 className="text-xl font-bold">POS</h1></div>
}
```

Create `apps/web/src/app/(stock)/stock/page.tsx`:
```tsx
export default function StockPage() {
  return <div className="p-6"><h1 className="text-xl font-bold">Stock</h1></div>
}
```

Create `apps/web/src/app/(admin)/admin/page.tsx`:
```tsx
export default function AdminPage() {
  return <div className="p-6"><h1 className="text-xl font-bold">Admin</h1></div>
}
```

- [ ] **Step 4: Start both apps and verify end-to-end auth flow**

Terminal 1 (API):
```bash
cd apps/api && pnpm dev
```

Terminal 2 (Web):
```bash
cd apps/web && pnpm dev
```

Open http://localhost:3000/login  
→ Login with `admin@merxylab.com` / `admin123`  
→ Should redirect to `/dashboard`  
→ Visit `/pos` — should also work (superadmin has all roles)  
→ Visit http://localhost:3000/login directly while logged in — verify redirect to dashboard

- [ ] **Step 5: Commit**

```bash
cd ../..
git add apps/web/middleware.ts apps/web/src/app apps/web/.env.local
git commit -m "feat: add Next.js middleware for role-based route protection"
```

---

## Task 15: Run all tests + push to GitHub

- [ ] **Step 1: Run all API tests**

```bash
cd apps/api && pnpm test
```

Expected: All tests pass. If coverage is below 80% on auth module, add missing test cases for `refreshToken` in `auth.service.spec.ts`:

```typescript
describe('refreshToken', () => {
  it('throws UnauthorizedException when token is invalid', async () => {
    mockJwt.verify.mockImplementation(() => { throw new Error('invalid') })
    await expect(service.refreshToken('bad-token')).rejects.toThrow(UnauthorizedException)
  })

  it('throws UnauthorizedException when token not in Redis', async () => {
    mockJwt.verify.mockReturnValue({ sub: 'u1', deviceId: 'd1', email: 'x@x.com', storeId: null, role: Role.SUPERADMIN })
    mockRedis.get.mockResolvedValue(null)
    await expect(service.refreshToken('some-token')).rejects.toThrow(UnauthorizedException)
  })
})
```

- [ ] **Step 2: Check test coverage**

```bash
cd apps/api && pnpm test:cov
```

Expected: auth module coverage ≥ 80%.

- [ ] **Step 3: Push to GitHub**

```bash
cd ../..
git push -u origin main
```

- [ ] **Step 4: Verify push succeeded**

```bash
git log --oneline -10
```

Expected: All Phase 1 commits visible.

---

## Phase 1 Complete

At this point you have:
- Turborepo monorepo with pnpm workspaces
- NestJS API (Fastify) with JWT auth, Redis refresh tokens, Prisma + PostgreSQL
- Shared `@merxy/types` and `@merxy/utils` packages
- Next.js 14 web app with Tailwind + design tokens
- Login page with multi-store selection
- Four panel layouts (POS, Dashboard, Stock, Admin)
- Route protection middleware
- All auth tests passing

**Next: Phase 2 — Stock & Products** (`docs/superpowers/plans/2026-04-18-phase-2-stock-products.md`)
