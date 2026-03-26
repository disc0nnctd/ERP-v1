# ERP Guardrails Skill

Use this skill whenever a change touches schema, auth, tRPC routers, services, validators, logging, patient data, visit data, appointment data, tenant boundaries, or any code that could expose PHI.

## Required Checks

1. Tenant isolation
   - Every new table includes `tenant_id`.
   - Every new query or mutation is scoped by trusted tenant context.
   - Never trust tenant identifiers from request params when session context exists.
2. Validation
   - All user input must be validated with Zod.
   - Derive types from Prisma and Zod instead of hand-writing DTOs.
3. PHI and logging
   - Never log patient data or other PHI.
   - Keep error messages safe for logs and telemetry.
4. Data lifecycle
   - Patient records are soft-deleted only.
   - Audit logging exists for mutations.
5. UI and API safety
   - No `dangerouslySetInnerHTML`.
   - No hardcoded secrets.
   - Auth is required on every non-public endpoint.
6. Verification
   - Add or update tenant isolation tests.
   - Run relevant typecheck, lint, and test commands.

## Project-Specific Rules

- Services own business logic; routers should stay thin.
- Shared validators belong in `src/lib/validators/`.
- Notifications must not block the request path.
- RBAC checks belong in the established permission and middleware flow.

## Skill Escalation

- If the task is a secure-coding review, prefer `security-best-practices` when available.
- If the task is explicit threat modeling, prefer `security-threat-model` when available.
- If the change affects both workflow and safety, use this skill together with `feature-delivery`.
