# Agents Guide - Hospital ERP Manager

This file is the canonical instruction set for coding agents in this repository.
Keep shared rules here. Keep tool-specific files thin.

## Canonical Sources

- `AGENTS.md` is the source of truth for repo-wide agent behavior.
- `CLAUDE.md` must stay thin and import this file rather than duplicate it.
- Repo skills should keep their canonical body in `docs/agent-skills/` and use thin wrappers in `.agents/skills/` and `.claude/skills/`.
- Add nested `AGENTS.md` files only when a subdirectory genuinely needs different rules.
- If a rule changes, update `AGENTS.md`, the relevant skill body, and the decision log in the same change.

## Agentic Development Standard

Every non-trivial feature follows this pipeline:

```text
1. PLAN -> 2. SPEC -> 3. TDD -> 4. IMPLEMENT -> 5. VERIFY -> 6. REVIEW -> 7. SECURITY -> 8. COMMIT
```

Operational rules:

- The agent that writes code is not the final certifier of that code.
- Manual dry run first, then automation. Do not schedule or background a workflow you have not run interactively.
- Prefer deterministic gates over prompt-only promises: tests, linters, typechecks, hooks, and explicit review steps.
- Parallel work is allowed only when write scopes are disjoint and each slice is independently verifiable.
- Use worktrees for parallel branches. Do not let multiple agents edit the same files concurrently.

## Skill-First Rules

Before writing a long custom prompt, check whether the task already matches a skill, command, subagent, or built-in workflow.

Skill priorities:

1. Repo skill
2. Personal skill
3. Built-in or system skill
4. Ad-hoc prompt

Use an existing skill when:

- The task is repetitive or procedural.
- The task needs reference docs, templates, scripts, or guardrails.
- The task should behave the same across Claude, Codex, and similar coding agents.

Create or update a skill when:

- The same workflow has repeated at least twice.
- The workflow spans enough steps that agents often skip or reorder them.
- The workflow depends on project-specific domain knowledge.
- You want a reusable slash command or explicit invocation target.

Skill design rules:

- Keep one job per skill. Do not create "do everything" skills.
- Put stable instructions in `docs/agent-skills/`.
- Keep tool-specific wrappers short and descriptive.
- State exactly when the skill should and should not trigger.
- Prefer instruction-only skills first. Add scripts only when determinism matters.

Current high-value skills to prefer when available in the session:

- `security-best-practices` for secure-by-default coding review.
- `security-threat-model` for explicit threat modeling.
- `openai-docs` for current OpenAI product guidance from official docs.
- `skill-creator` when packaging a repeatable workflow into a new skill.
- `skill-installer` when a useful skill exists elsewhere and should be installed instead of recreated.

Current repo skills:

- `feature-delivery`
- `erp-guardrails`

## Tool-Specific Usage

### Claude Code

- Shared memory lives in `CLAUDE.md`, which should import this file.
- Project skills live in `.claude/skills/`.
- Project subagents, if added, live in `.claude/agents/`.
- Project hooks live in `.claude/settings.json`.
- Local machine-only settings belong in `.claude/settings.local.json` and must not be committed.
- Built-in Claude skills worth preferring for large workflows include `/batch`, `/simplify`, `/loop`, and `/agents` when they fit the job.

### Codex

- Repo instructions live in `AGENTS.md`.
- Repo skills live in `.agents/skills/`.
- Codex may also use nested `AGENTS.md` files and nested `.agents/skills/` directories.
- Prefer `/skills`, `$skill-creator`, and `$skill-installer` before inventing new workflow text.
- Use subagents only when the task is explicitly parallelizable or benefits from isolated context.
- Use automations only after an interactive dry run proves the prompt, tools, and resulting diff are safe.

### Cross-Tool Rule

- Shared workflow and domain knowledge should be tool-agnostic first, tool-specific second.
- If a feature is supported by both Claude and Codex, document the canonical behavior once and wrap it for each tool.
- Avoid copy-pasting large instruction blocks between `AGENTS.md`, `CLAUDE.md`, and skill wrappers.

## Workflow Roles

| Step | Role | What It Does | Trigger |
|------|------|--------------|---------|
| 1 | `planner` | Breaks work into atomic, verifiable tasks and dependencies | Start of any non-trivial feature |
| 2 | `architect` | Designs schema, API contracts, and component boundaries | New module or architecture change |
| 3 | `tdd-guide` | Writes failing tests first and defines verification scope | Before implementation |
| 4 | `developer` | Implements the minimal diff to satisfy tests and acceptance criteria | After tests are written |
| 5 | `code-reviewer` | Reviews behavior, regressions, maintainability, and edge cases | After implementation |
| 6 | `security-reviewer` | Reviews tenant isolation, PHI exposure, validation, and auth | Before commit |
| 7 | `build-error-resolver` | Fixes type/build failures with minimal blast radius | When `pnpm build` or `pnpm typecheck` fails |
| 8 | `doc-updater` | Keeps docs, skills, and decision logs aligned with reality | When process or patterns change |

Role rules:

- `planner`, `architect`, `code-reviewer`, and `security-reviewer` should usually be read-heavy, not write-heavy.
- `developer` and `build-error-resolver` should prefer the smallest workable diff.
- `doc-updater` must keep `CLAUDE.md` thin and move durable guidance into `AGENTS.md` or `docs/agent-skills/`.
- `code-reviewer` and `security-reviewer` should be independent gates, not rubber stamps from the builder.

## Role Definitions

### planner

Purpose: Break features into implementation steps before coding starts.

When:

- Any feature touching 3 or more files
- New modules
- Unclear scope
- Any work that could benefit from parallelization

Output:

- Ordered list of atomic tasks
- File paths
- Dependencies
- Acceptance criteria
- Flags for architecture or security review

Instructions:

- Read the current sprint scope from the Progress Tracker.
- Reference existing patterns before proposing new ones.
- Keep tasks independently testable.
- Separate research, test writing, implementation, and review work.

### architect

Purpose: System design decisions for schema, API contracts, tenant isolation, and RBAC.

When:

- New entity or table
- New module
- Changes to tenant isolation
- New third-party integration
- Any change that affects long-term patterns

Output:

- Prisma schema changes
- API procedure signatures
- Validator changes
- Component hierarchy
- Risks and tradeoffs

Instructions:

- Every table must include `tenant_id`.
- Types must derive from Prisma and Zod, not hand-written DTOs.
- Follow existing patterns in `src/server/trpc/routers/` and `src/server/services/`.
- Consider Phase 2 AI readiness: structured timestamps, machine-friendly fields, and flexible JSON where justified.

### tdd-guide

Purpose: Write tests before implementation and enforce RED -> GREEN -> REFACTOR.

When:

- Every new service method
- Every new tRPC procedure
- Every bug fix with a reproducible symptom
- Critical UI interactions

Output:

- Failing tests first
- Clear statement of what behavior is now covered

Instructions:

- Unit tests: `src/server/services/*.test.ts`
- Router tests: `src/server/trpc/routers/*.test.ts`
- E2E tests: `e2e/*.spec.ts` for critical flows only
- Target 80 percent plus coverage on business logic
- Test tenant isolation explicitly

### developer

Purpose: Implement the smallest change that satisfies the tests, spec, and guardrails.

Instructions:

- Follow the spec before improvising.
- Prefer existing patterns over novel abstractions.
- Keep diffs reviewable.
- Run the most relevant verification after each logical step.
- Do not claim success without evidence from commands or tests.

### code-reviewer

Purpose: Post-implementation quality review.

Checklist:

- [ ] Functions stay small and readable
- [ ] No `any` types
- [ ] Error handling is explicit
- [ ] No accidental behavior regressions
- [ ] Tenant scoping is visible on every new query
- [ ] Audit logging exists on mutations
- [ ] Tests cover the changed behavior
- [ ] Files stay appropriately sized

### security-reviewer

Purpose: Security review before code enters the repository.

Checklist:

- [ ] No secrets in code
- [ ] All user inputs validated with Zod
- [ ] No PHI in logs or error text
- [ ] Auth required on all non-public endpoints
- [ ] Tenant isolation is impossible to bypass in normal code paths
- [ ] SQL injection prevented through Prisma or equivalent safe APIs
- [ ] No unsafe HTML rendering
- [ ] Soft-delete is preserved for patient records
- [ ] CSRF protections remain intact

### build-error-resolver

Purpose: Fix build or type failures with minimal changes.

Instructions:

- Fix the root cause, not the symptom.
- No `@ts-ignore`.
- No `as any`.
- Run `pnpm typecheck` after each fix.
- If a Prisma type error appears, run `pnpm db:generate` first.

### doc-updater

Purpose: Keep repo guidance aligned with reality.

Instructions:

- Update `AGENTS.md`, `CLAUDE.md`, skill bodies, and decision logs together when process changes.
- Record new gotchas.
- Update canonical examples when they become real.
- Keep `CLAUDE.md` under 80 lines.

## Project Architecture Rules

- Multi-tenancy: every table has `tenant_id`. Never query without tenant context.
- Types are derived, never hand-written: Prisma for DB types, Zod for validation, tRPC for end-to-end inference.
- Server Components by default. Add `'use client'` only when interactivity requires it.
- Routers live in `src/server/trpc/routers/`, one file per domain.
- Services live in `src/server/services/`.
- Shared validators live in `src/lib/validators/`.
- No barrel files in `src/app/`.

## Code Style

- Named exports only, except where Next.js requires default exports.
- Prefer immutable patterns.
- Prefer `satisfies` over broad type assertions.
- Handle errors explicitly.
- Avoid `any`; use `unknown` with type guards.
- Use camelCase for variables and functions, PascalCase for types and components, kebab-case for files, and UPPER_SNAKE_CASE for constants.

## Project Guardrails

- Never hardcode secrets. Validate env vars at startup.
- Never log PHI in console output, telemetry, or error strings.
- Never skip `tenant_id` in a new table.
- Never hard-delete patient data.
- Never read tenant identity from request params when trusted session context exists.
- Never block the request/response path on fire-and-forget notifications.
- Never add features outside the current sprint without approval.

## Verification Gates

Run after every meaningful change:

1. `pnpm typecheck`
2. `pnpm lint`
3. Relevant tests only, not the entire suite by default
4. After Prisma schema changes: `pnpm db:generate` then `pnpm typecheck`

Additional gate rules:

- Builder output is not complete until at least one deterministic verification step has run.
- Review and security gates happen after implementation, not instead of implementation verification.
- For automation, inspect the first few runs manually before trusting the workflow.

## Specs

Before implementing a non-trivial feature, write a spec in `docs/specs/` using this template:

```markdown
# docs/specs/[feature-name].md

## Overview
What this feature does and why.

## User Stories
- As a [role], I want to [action] so that [benefit]

## Data Model Changes
Prisma schema additions or modifications.

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

## Automation And Parallelism

Best-practice rules for agentic automation:

- Dry-run prompts interactively before scheduling or backgrounding them.
- Use hooks for deterministic guardrails such as blocking destructive commands, scanning for PHI in logs, or triggering targeted verification.
- Default unattended work to the least-permissive sandbox that still works.
- Use worktrees for parallel feature slices, reviews, or experiments.
- Give each parallel task a clear file ownership boundary.
- Initialize dependencies and environment per worktree when needed.
- Review the first few automation runs closely and adjust prompt or cadence before trusting them.

Suggested use in this repo:

- Worktree per feature slice or review lane
- Independent review and security lane after implementation
- Skills for repeatable workflows
- Hooks for guardrails, not for replacing tests

## Context Management Rules

1. Start fresh between unrelated sprints or features.
2. Reset after two failed correction attempts and restate the problem cleanly.
3. Use skills or read-only subagents for investigation so the main context stays focused.
4. Keep one feature per session when possible.
5. Reference files by path instead of pasting large code blocks.
6. Keep specs, plans, and acceptance criteria in files, not only in chat.
7. Split large instruction files into nested files or skills instead of letting root guidance bloat.

## Git Workflow

### Branch Naming

```text
feat/[sprint]-[feature]
fix/[sprint]-[description]
refactor/[description]
```

### Commit Format

```text
<type>: <description>

<optional body explaining why>
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`

No Claude or Codex attribution in commits.

### PR Checklist

Before creating a PR, verify:

- [ ] `pnpm typecheck` passes
- [ ] `pnpm lint` passes
- [ ] Relevant tests pass
- [ ] Code review completed
- [ ] Security review completed
- [ ] No PHI in logs or error messages
- [ ] Tenant isolation tested for new queries
- [ ] Specs and docs updated if behavior changed

## Progress Tracker

### Phase 1 - Core ERP

#### Sprint 1: Foundation (Weeks 1-2)
- [ ] Project scaffolding (Next.js + TypeScript + Tailwind + shadcn/ui)
- [ ] Biome config, tsconfig strict, husky + lint-staged
- [ ] Prisma schema - all entities, enums, relations, indexes
- [ ] Tenant-scoped Prisma middleware
- [ ] Auth: signup (create tenant + first user)
- [ ] Auth: login / logout / session with tenantId + role in JWT
- [ ] Layout shell: sidebar, header, tenant context provider
- [ ] Environment validation (Zod schema for env vars)

#### Sprint 2: Patient Management (Weeks 3-4)
- [ ] Patient CRUD - create, read, update (soft-delete)
- [ ] Patient search - PostgreSQL full-text (tsvector + GIN index)
- [ ] UserPatient tagging - auto-tag on create, manual re-tag
- [ ] Duplicate detection - same name + DOB + phone warning
- [ ] Patient list page - searchable, paginated table
- [ ] Patient detail page - demographics, visits, vitals tabs
- [ ] New patient registration form with Zod validation
- [ ] Audit logging middleware on all patient mutations

#### Sprint 3: Appointment Scheduling (Weeks 5-6)
- [ ] Doctor availability - weekly recurring slots CRUD
- [ ] Date blocking - vacation and holiday overrides
- [ ] Slot generation - available slots for doctor and date
- [ ] Scheduled booking - with conflict detection
- [ ] Walk-in queue - auto-incrementing position
- [ ] Appointment status lifecycle - state machine transitions
- [ ] Calendar view (day and week) for doctors
- [ ] List view for receptionists
- [ ] Booking form - select patient, doctor, date, slot

#### Sprint 4: Visits And Notifications (Weeks 7-8)
- [ ] Visit creation from appointment
- [ ] SOAP notes form (4 separate fields)
- [ ] Vitals recording - BP, temp, weight, height, pulse, SpO2
- [ ] Prescription field + file attachments (S3 or R2)
- [ ] Twilio integration - SMS + WhatsApp
- [ ] Reminder job scheduling - 24hr + 1hr before appointment
- [ ] Cancel pending reminders on appointment cancellation
- [ ] Notification status tracking + retry (max 3)

#### Sprint 5: Polish And Launch (Weeks 9-10)
- [ ] User invitation flow - email invite with role assignment
- [ ] Accept invite page - set password, join tenant
- [ ] RBAC-based UI - show or hide features per role
- [ ] Dashboard - today's appointments, patient count, quick stats
- [ ] Error handling + loading states across all pages
- [ ] Rate limiting on auth endpoints
- [ ] Security hardening pass
- [ ] Deployment pipeline - Vercel + DB provisioning
- [ ] Smoke tests for critical paths (E2E)

### Phase 2 - AI Layer (Future)
- [ ] Smart scheduling suggestions
- [ ] Auto-generated visit summaries (LLM)
- [ ] No-show prediction model
- [ ] Voice-to-notes (speech-to-text + LLM SOAP structuring)
- [ ] Drug interaction alerts
- [ ] Patient intake chatbot (WhatsApp)

## Canonical Examples

Once implemented, these files serve as the pattern to follow for new additions:

| Pattern | Reference File | Notes |
|---------|----------------|-------|
| tRPC router | `src/server/trpc/routers/patient.ts` | Query + mutation structure |
| Service layer | `src/server/services/patient.service.ts` | Business logic, tenant scoping |
| Zod validator | `src/lib/validators/patient.ts` | Shared FE and BE schema |
| List page | `src/app/(dashboard)/patients/page.tsx` | Server component + table |
| Detail page | `src/app/(dashboard)/patients/[id]/page.tsx` | Tabs layout |
| Form component | `src/components/forms/PatientForm.tsx` | React Hook Form + Zod |

Update these references as files are created during Sprint 2.

## Decision Log

| Date | Decision | Rationale | Affects |
|------|----------|-----------|---------|
| 2026-03-26 | Shared DB multi-tenancy with tenant_id column | Cost-effective for MVP, 5-10 users per tenant | All tables, Prisma middleware |
| 2026-03-26 | Biome over ESLint + Prettier | Faster, single config, greenfield project | Formatting and linting |
| 2026-03-26 | tRPC over REST | End-to-end type safety with Prisma + Zod | All API procedures |
| 2026-03-26 | RBAC in code for Phase 1 | Simpler while roles are limited | `permissions.ts`, middleware |
| 2026-03-26 | SOAP notes as four separate fields | Better for future extraction and summarization | Visit schema, visit form |
| 2026-03-26 | Soft-delete only for patient data | Compliance and audit trail | Patient model, patient queries |
| 2026-03-26 | `AGENTS.md` is the shared source of truth and `CLAUDE.md` stays thin | Prevents instruction drift across tools | Agent guidance, repo maintenance |
| 2026-03-26 | Skill-first workflow with canonical bodies in `docs/agent-skills/` | Reusable workflows should outlive chat sessions and work across tools | Claude skills, Codex skills, repo automation |
| 2026-03-26 | Parallel agent work requires isolated worktrees and independent certification | Reduces context rot, merge conflicts, and self-certification failures | Large features, reviews, automation |
