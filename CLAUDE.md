# Hospital ERP Manager

Multi-tenant SaaS for small clinics. Next.js 14+ App Router, TypeScript, PostgreSQL, Prisma, tRPC, shadcn/ui, Tailwind.

## Commands

```bash
pnpm dev              # Start dev server
pnpm build            # Production build
pnpm typecheck        # TypeScript strict check
pnpm lint             # Biome lint + format check
pnpm lint:fix         # Biome auto-fix
pnpm test             # Vitest unit/integration tests
pnpm test:e2e         # Playwright E2E tests
pnpm db:generate      # Prisma client generation (after schema changes)
pnpm db:push          # Push schema to DB
pnpm db:migrate       # Create + apply migration
pnpm db:seed          # Seed dev data
```

## Architecture Rules

- Multi-tenancy: every table has `tenant_id`. Prisma middleware auto-scopes all queries. NEVER query without tenant context.
- Types are derived, never hand-written: Prisma generates DB types, Zod schemas validate inputs, tRPC infers end-to-end.
- Server Components by default. Add `'use client'` only when hooks/interactivity are needed.
- tRPC routers in `src/server/trpc/routers/` — one file per domain. Follow `src/server/trpc/routers/patient.ts` as canonical example.
- Services in `src/server/services/` — business logic lives here, not in routers.
- Zod validators in `src/lib/validators/` — shared between frontend forms and backend procedures.
- No barrel files (`index.ts` re-exports) in `src/app/` directories.

## Code Style

- Named exports only, no default exports (except Next.js pages/layouts which require them)
- Immutable patterns: always return new objects, never mutate parameters
- `satisfies` operator for type narrowing over `as` assertions
- Explicit error handling at every level — no silent catches
- Functions < 50 lines, files < 400 lines (800 max)
- No `any` — use `unknown` + type guards instead
- camelCase functions/variables, PascalCase types/components, kebab-case files, UPPER_SNAKE constants

## Verification (run after every change)

1. `pnpm typecheck` — must pass with zero errors
2. `pnpm lint` — must pass
3. Run relevant test file, not full suite: `pnpm test src/server/services/patient.test.ts`
4. After Prisma schema changes: `pnpm db:generate` then `pnpm typecheck`

## Key Patterns

- **RBAC**: role-permission map in `src/lib/permissions.ts`. Check via tRPC middleware `hasPermission('patients:create')`.
- **Audit logging**: automatic via tRPC middleware on all mutations. Never skip.
- **Soft-delete**: patients use `deleted_at` timestamp. Never hard-delete patient data.
- **Appointment status**: BOOKED -> CHECKED_IN -> IN_PROGRESS -> COMPLETED. Validate transitions in service layer.
- **Notifications**: fire-and-forget via job queue. Never block request on notification delivery.

## Gotchas

- tRPC context provides `session.tenantId` — always use this, never read tenant from request params
- Prisma `$extends` for tenant middleware must be applied before any query in the request lifecycle
- NextAuth v5 session callback must embed `tenantId` and `role` into the JWT
- Walk-in appointments skip slot validation — they get `type: WALK_IN` and auto queue position
- PostgreSQL full-text search uses `tsvector` — requires a GIN index migration, not a code change

## Do NOT

- Hardcode secrets — use env vars via `src/lib/env.ts` (validated with Zod at startup)
- Log patient data (PHI) — audit log only, never console/Sentry
- Skip tenant_id in any new table — every row must be tenant-scoped
- Create microservices or separate backends — this is a monorepo, single deployable
- Add features beyond the current sprint scope without explicit approval
