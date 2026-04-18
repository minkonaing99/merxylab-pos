# MerxyLab POS

Multi-store POS web app for Myanmar market.

## Key Conventions

- **Currency:** Myanmar Kyat — always display as `Ks`, store as integers (no floats)
- **Green** (`#059669`) only for: CTAs, active nav, positive KPI deltas. Everything else uses slate/charcoal.
- **Touch targets:** 44×44px minimum. Form inputs `font-size: 16px` (prevents iOS zoom).
- **Stock mutations:** always atomic — `StockMovement` row + `StoreProduct.stockQty` in one Prisma `$transaction`.
- **JWT scope:** `storeId` comes from the JWT claim — never from the request body.

## Rules

@rules/typescript/coding-style.md
@rules/typescript/patterns.md
@rules/typescript/testing.md
@rules/typescript/security.md

## Docs

@docs/design.md
@docs/phase-1.md
@docs/phase-2.md
@docs/phase-3.md
@docs/phase-4.md
@docs/phase-5.md
@docs/phase-6.md
@docs/phase-7.md
