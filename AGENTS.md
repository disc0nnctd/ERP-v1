# Agents Guide — Hospital ERP Manager

This document defines the AI agent workflow for developing this project. It serves as both a reference for which agents to use and a living progress tracker.

---

## Agent Workflow Per Feature

Every feature follows this pipeline. Agents are invoked in order — skip none.

```
1. PLAN ──> 2. SPEC ──> 3. TDD ──> 4. IMPLEMENT ──> 5. REVIEW ──> 6. SECURITY ──> 7. COMMIT
```

| Step | Agent | What It Does | Trigger |
|------|-------|-------------|---------|
| 1 | `planner` | Break feature into atomic tasks, identify dependencies | Start of any non-trivial feature |
| 2 | `architect` | Design data model, API contracts, component structure | New module or architectural decision |
| 3 | `tdd-guide` | Write tests first (RED), verify they fail | Before any implementation code |
| 4 | *developer* | Write minimal code to pass tests (GREEN), then refactor | After tests are written |
| 5 | `code-reviewer` | Review for quality, patterns, edge cases | After implementation |
| 6 | `security-reviewer` | Check for OWASP top 10, PHI leaks, tenant isolation | Before commit |
| 7 | *commit* | Conventional commit with descriptive message | After all checks pass |

### Parallel Agent Usage

Independent concerns run in parallel — never sequentially:

```
GOOD:  Launch code-reviewer + security-reviewer + tdd-guide simultaneously
BAD:   Wait for code-reviewer, then security-reviewer, then tdd-guide
```

### When to Use Subagents vs Direct Work

| Scenario | Approach |
|----------|----------|
| Investigate how an existing module works | Subagent (Explore type) |
| Review code after implementation | Subagent (code-reviewer) |
| Write a new tRPC router | Direct work (follow canonical example) |
| Debug a failing test | Direct work first, subagent if stuck after 2 attempts |
| Security audit before merge | Subagent (security-reviewer) |
| Research a library or pattern | Subagent (general-purpose) |

---

## Agent Definitions

### planner

**Purpose**: Break features into implementation steps before coding starts.

**When**: Any feature touching 3+ files, new modules, or unclear scope.

**Output**: Ordered list of atomic tasks (1-2 hours each) with file paths, dependencies, and acceptance criteria.

**Instructions**:
- Read the current sprint scope from the Progress Tracker below
- Reference existing patterns in the codebase
- Each task must be independently verifiable
- Flag tasks that need architectural decisions

### architect

**Purpose**: System design decisions — data model, API contracts, multi-tenancy, RBAC.

**When**: New entity/table, new module, changes to tenant isolation, new integration.

**Output**: Data model changes (Prisma schema diff), API procedure signatures, component hierarchy.

**Instructions**:
- Every table must include `tenant_id` — no exceptions
- Types derived from Prisma + Zod, never hand-written
- Follow existing patterns in `src/server/trpc/routers/` and `src/server/services/`
- Consider Phase 2 AI readiness (timestamps, structured data, JSONB flexibility)

### tdd-guide

**Purpose**: Write tests before implementation. Enforce RED -> GREEN -> REFACTOR.

**When**: Every new function, service method, tRPC procedure, and UI interaction.

**Output**: Test files that initially FAIL, then pass after implementation.

**Instructions**:
- Unit tests: `src/server/services/*.test.ts` (Vitest)
- Router tests: `src/server/trpc/routers/*.test.ts` (Vitest + tRPC caller)
- E2E tests: `e2e/*.spec.ts` (Playwright) for critical user flows only
- Target 80%+ coverage on business logic (services, validators)
- Test tenant isolation explicitly: verify user A cannot access tenant B's data

### code-reviewer

**Purpose**: Post-implementation quality review.

**When**: After every feature implementation, before commit.

**Checklist**:
- [ ] Functions < 50 lines, files < 400 lines
- [ ] No `any` types — `unknown` + type guards used
- [ ] Immutable patterns — no parameter mutation
- [ ] Error handling at every level
- [ ] No hardcoded values — constants or env vars used
- [ ] Tenant scoping verified on new queries
- [ ] Audit logging present on mutations
- [ ] No barrel files in `src/app/`

### security-reviewer

**Purpose**: Security audit before code enters the repository.

**When**: Before every commit. Mandatory for auth, patient data, and API endpoints.

**Checklist**:
- [ ] No secrets in code (API keys, passwords, connection strings)
- [ ] All user inputs validated via Zod schemas
- [ ] SQL injection prevented (Prisma parameterized queries only)
- [ ] XSS prevented (React auto-escaping, no `dangerouslySetInnerHTML`)
- [ ] Tenant isolation: no cross-tenant data access possible
- [ ] PHI not in logs (console, Sentry, or error messages)
- [ ] Auth required on all non-public endpoints
- [ ] Rate limiting on auth endpoints
- [ ] Soft-delete used for patient records (no hard delete)
- [ ] CSRF protection via SameSite cookies

### build-error-resolver

**Purpose**: Fix build/type errors with minimal changes.

**When**: `pnpm build` or `pnpm typecheck` fails.

**Instructions**:
- Fix the error, not the symptom — no `@ts-ignore` or `as any`
- Minimal diff — don't refactor surrounding code
- Run `pnpm typecheck` after each fix to verify
- If a Prisma type error: run `pnpm db:generate` first

### doc-updater

**Purpose**: Keep this AGENTS.md and CLAUDE.md in sync with reality.

**When**: After completing a sprint or when patterns/conventions change.

**Instructions**:
- Update the Progress Tracker below
- Add new gotchas discovered during development
- Update canonical example references if they change
- Keep CLAUDE.md under 80 lines — move details here

---

## Spec-Driven Development

Before implementing a feature, write a spec in `docs/specs/`:

```markdown
# docs/specs/[feature-name].md

## Overview
What this feature does and why.

## User Stories
- As a [role], I want to [action] so that [benefit]

## Data Model Changes
Prisma schema additions/modifications.

## API Procedures
| Procedure | Input (Zod) | Output | Auth |
|-----------|-------------|--------|------|

## UI Components
Pages and components needed.

## Edge Cases
- What happens when...
- What if...

## Acceptance Criteria
- [ ] Verifiable condition 1
- [ ] Verifiable condition 2

## Test Plan
- Unit: what to test
- Integration: what to test
- E2E: critical flow to test
```

---

## Context Management Rules

AI context is the scarcest resource. Follow these rules:

1. **`/clear` between sprints** — start fresh for each sprint
2. **`/clear` after 2 failed correction attempts** — restate the problem cleanly
3. **Use subagents for investigation** — don't pollute main context with exploration
4. **One feature per session** — avoid mixing unrelated work
5. **Reference files by path** — don't paste large code blocks into chat
6. **`/compact` with focus hint** — e.g., `/compact Focus on appointment booking logic`

---

## Git Workflow

### Branch Naming
```
feat/[sprint]-[feature]     # feat/s1-auth-signup
fix/[sprint]-[description]  # fix/s2-patient-search-index
refactor/[description]      # refactor/tenant-middleware
```

### Commit Format
```
<type>: <description>

<optional body explaining why>
```
Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`

No Claude attribution in commits (per global CLAUDE.md).

### PR Checklist
Before creating a PR, verify:
- [ ] `pnpm typecheck` passes
- [ ] `pnpm lint` passes
- [ ] `pnpm test` passes (relevant tests)
- [ ] Security review completed (subagent)
- [ ] Code review completed (subagent)
- [ ] No PHI in logs or error messages
- [ ] Tenant isolation tested for new queries
- [ ] Progress Tracker updated below

---

## Progress Tracker

### Phase 1 — Core ERP

#### Sprint 1: Foundation (Weeks 1-2)
- [ ] Project scaffolding (Next.js + TypeScript + Tailwind + shadcn/ui)
- [ ] Biome config, tsconfig strict, husky + lint-staged
- [ ] Prisma schema — all entities, enums, relations, indexes
- [ ] Tenant-scoped Prisma middleware
- [ ] Auth: signup (create tenant + first user)
- [ ] Auth: login / logout / session with tenantId + role in JWT
- [ ] Layout shell: sidebar, header, tenant context provider
- [ ] Environment validation (Zod schema for env vars)

#### Sprint 2: Patient Management (Weeks 3-4)
- [ ] Patient CRUD — create, read, update (soft-delete)
- [ ] Patient search — PostgreSQL full-text (tsvector + GIN index)
- [ ] UserPatient tagging — auto-tag on create, manual re-tag
- [ ] Duplicate detection — same name + DOB + phone warning
- [ ] Patient list page — searchable, paginated table
- [ ] Patient detail page — demographics, visits, vitals tabs
- [ ] New patient registration form with Zod validation
- [ ] Audit logging middleware on all patient mutations

#### Sprint 3: Appointment Scheduling (Weeks 5-6)
- [ ] Doctor availability — weekly recurring slots CRUD
- [ ] Date blocking — vacation/holiday overrides
- [ ] Slot generation — available slots for doctor + date
- [ ] Scheduled booking — with conflict detection
- [ ] Walk-in queue — auto-incrementing position
- [ ] Appointment status lifecycle — state machine transitions
- [ ] Calendar view (day/week) for doctors
- [ ] List view for receptionists
- [ ] Booking form — select patient, doctor, date, slot

#### Sprint 4: Visits & Notifications (Weeks 7-8)
- [ ] Visit creation from appointment
- [ ] SOAP notes form (4 separate fields)
- [ ] Vitals recording — BP, temp, weight, height, pulse, SpO2
- [ ] Prescription field + file attachments (S3/R2)
- [ ] Twilio integration — SMS + WhatsApp
- [ ] Reminder job scheduling — 24hr + 1hr before appointment
- [ ] Cancel pending reminders on appointment cancellation
- [ ] Notification status tracking + retry (max 3)

#### Sprint 5: Polish & Launch (Weeks 9-10)
- [ ] User invitation flow — email invite with role assignment
- [ ] Accept invite page — set password, join tenant
- [ ] RBAC-based UI — show/hide features per role
- [ ] Dashboard — today's appointments, patient count, quick stats
- [ ] Error handling + loading states across all pages
- [ ] Rate limiting on auth endpoints
- [ ] Security hardening pass (full security-reviewer audit)
- [ ] Deployment pipeline — Vercel + DB provisioning
- [ ] Smoke tests for critical paths (E2E)

### Phase 2 — AI Layer (Future)
- [ ] Smart scheduling suggestions
- [ ] Auto-generated visit summaries (LLM)
- [ ] No-show prediction model
- [ ] Voice-to-notes (speech-to-text + LLM SOAP structuring)
- [ ] Drug interaction alerts
- [ ] Patient intake chatbot (WhatsApp)

---

## Canonical Examples

Once implemented, these files serve as the pattern to follow for new additions:

| Pattern | Reference File | Notes |
|---------|---------------|-------|
| tRPC router | `src/server/trpc/routers/patient.ts` | Query + mutation structure |
| Service layer | `src/server/services/patient.service.ts` | Business logic, tenant scoping |
| Zod validator | `src/lib/validators/patient.ts` | Shared FE/BE schema |
| List page | `src/app/(dashboard)/patients/page.tsx` | Server component + table |
| Detail page | `src/app/(dashboard)/patients/[id]/page.tsx` | Tabs layout |
| Form component | `src/components/forms/PatientForm.tsx` | React Hook Form + Zod |

*Update these references as files are created during Sprint 2.*

---

## Decision Log

Track architectural decisions that affect future work:

| Date | Decision | Rationale | Affects |
|------|----------|-----------|---------|
| 2026-03-26 | Shared DB multi-tenancy with tenant_id column | Cost-effective for MVP, 5-10 users/tenant | All tables, Prisma middleware |
| 2026-03-26 | Biome over ESLint+Prettier | 10-25x faster, single config, greenfield project | All code formatting/linting |
| 2026-03-26 | tRPC over REST | End-to-end type safety with Prisma + Zod | All API procedures |
| 2026-03-26 | RBAC in code (not DB) for Phase 1 | Simpler, only 2 roles; migrate to DB when roles grow | permissions.ts, tRPC middleware |
| 2026-03-26 | SOAP notes as 4 separate fields | Enables Phase 2 LLM extraction/summarization | Visit schema, visit form |
| 2026-03-26 | Soft-delete only for patient data | Regulatory compliance, audit trail | Patient model, all queries |
