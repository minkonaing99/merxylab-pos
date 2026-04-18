# MerxyLab POS

Summary: Multi-store Point-of-Sale web app for the Myanmar market with four panels (Cashier POS, Dashboard, Admin, Stock), per-store inventory, RBAC, and real-time sync across terminals. Currency is Myanmar Kyat (Ks).

## Commands
- dev: `pnpm dev`
- build: `pnpm build`
- test: `pnpm test`
- lint: `pnpm lint`
- migrate: `cd apps/api && npx prisma migrate dev`
- generate: `cd apps/api && npx prisma generate`

## Architecture
- `apps/web/`: Next.js 14 App Router frontend — four route groups: `(pos)`, `(dashboard)`, `(stock)`, `(admin)`
- `apps/api/`: NestJS + Fastify backend — modules: auth, products, inventory, orders, shifts, refunds, reports, admin
- `packages/types/`: Shared TypeScript interfaces and enums used by both apps
- `packages/utils/`: Shared pure utility functions (formatters, validators)
- `docs/`: Design spec (`design.md`) and phase implementation plans (`phase-1.md` → `phase-7.md`)
- Entry: web calls api over REST + Socket.IO; api talks to PostgreSQL via Prisma and Redis via ioredis

## Code Style
- Language: TypeScript strict (both apps)
- Formatter: Prettier
- Linter: ESLint
- Monorepo: Turborepo + pnpm workspaces
- Currency: always integers in Ks — never floats, never `$`
- Green (`#059669`) only for CTAs, active nav, positive deltas — not for general UI
- Touch targets: 44×44px min; form inputs `font-size: 16px` (prevents iOS auto-zoom)

## Rules
- Never commit .env files.
- Never modify migrations directly.
- Never install packages without asking.
- Never push to main without tests passing.
- Never use `any` type — create proper types in `packages/types`.
- Never delete user data without confirmation.
- Never expose JWT secrets or S3 keys in client code.
- Never skip Zod validation on API inputs.
- Never mutate stock outside a Prisma `$transaction` (StockMovement + StoreProduct.stockQty must update atomically).
- Never resolve `storeId` from the request body — always use the JWT claim.

## Lessons Learned
- None yet. Append here when mistakes happen so they are not repeated.
