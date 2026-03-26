# Hospital ERP Manager

A multi-tenant SaaS platform for small clinics and hospitals to manage operations with minimal staff. Built for doctor-owned practices with 5-10 users per clinic.

---

## Problem

Small clinics depend heavily on manual coordination — paper registers, phone-based appointment booking, and receptionist-driven workflows. This creates bottlenecks, errors, and a need for more staff than necessary.

## Solution

A focused ERP that digitizes the two highest-impact workflows first — **patient registration** and **appointment scheduling** — with an extensible architecture that supports AI features in the future.

---

## Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | Next.js 14+ (App Router) | SSR/SSG, colocated API routes, single deployable |
| Language | TypeScript (strict mode) | End-to-end type safety |
| UI | shadcn/ui + Tailwind CSS | Accessible, customizable, no vendor lock-in |
| API | tRPC | Type-safe RPC, zero code generation |
| ORM | Prisma | Type-safe queries, migrations, excellent DX |
| Database | PostgreSQL | JSONB flexibility, full-text search, row-level security |
| Auth | NextAuth.js v5 (Auth.js) | Multi-provider, session management |
| Validation | Zod | Shared schemas between frontend and backend |
| Background Jobs | Inngest / BullMQ | Notification scheduling, retries |
| Notifications | Twilio (SMS + WhatsApp) | Single vendor for both channels |
| File Storage | AWS S3 / Cloudflare R2 | Prescription uploads, lab reports |
| Linting | Biome | Fast, single config for lint + format |
| Testing | Vitest + Playwright | Unit/integration + E2E |
| Hosting | Vercel + Supabase/AWS RDS | Low-ops for a startup |

---

## Architecture

### Multi-Tenancy

Shared database, shared schema, `tenant_id` column on every table. Prisma middleware auto-injects `WHERE tenant_id = ?` on every query.

### Project Structure

```
hospital-erp/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
├── src/
│   ├── app/                        # Next.js App Router
│   │   ├── (auth)/                 # Login, signup, invite
│   │   ├── (dashboard)/            # Authenticated pages
│   │   │   ├── patients/
│   │   │   ├── appointments/
│   │   │   ├── schedule/
│   │   │   └── settings/
│   │   └── api/trpc/[trpc]/
│   ├── server/
│   │   ├── trpc/routers/           # One router per domain
│   │   ├── services/               # Business logic
│   │   ├── db/                     # Prisma client + tenant middleware
│   │   └── jobs/                   # Background job processors
│   ├── lib/
│   │   ├── validators/             # Zod schemas (shared FE/BE)
│   │   ├── permissions.ts          # RBAC role-permission map
│   │   └── env.ts                  # Env var validation
│   ├── components/
│   │   ├── ui/                     # shadcn/ui primitives
│   │   ├── forms/
│   │   ├── tables/
│   │   └── layout/
│   └── hooks/
├── docs/specs/                     # Feature specs (spec-driven dev)
├── e2e/                            # Playwright E2E tests
├── biome.json
├── tsconfig.json
└── package.json
```

---

## Data Model

```mermaid
erDiagram
    Tenant ||--o{ User : "has many"
    Tenant ||--o{ Patient : "has many"
    Tenant ||--o{ Appointment : "has many"
    Tenant ||--o{ AuditLog : "has many"

    User ||--o{ UserPatient : "manages"
    Patient ||--o{ UserPatient : "managed by"

    User ||--o{ DoctorAvailability : "has slots"
    User ||--o{ Appointment : "assigned doctor"
    Patient ||--o{ Appointment : "has appointments"
    Patient ||--o{ Visit : "has visits"
    Visit ||--o{ Vitals : "has vitals"

    Appointment ||--o| Visit : "produces"
    Appointment ||--o{ Notification : "triggers"

    Tenant {
        uuid id PK
        string name
        string slug
        jsonb settings
    }

    User {
        uuid id PK
        uuid tenant_id FK
        string email
        string name
        enum role
        boolean is_active
    }

    Patient {
        uuid id PK
        uuid tenant_id FK
        string first_name
        string last_name
        date date_of_birth
        enum gender
        string phone
        text allergies
        text chronic_conditions
        datetime deleted_at
    }

    UserPatient {
        uuid id PK
        uuid user_id FK
        uuid patient_id FK
        uuid tenant_id FK
    }

    DoctorAvailability {
        uuid id PK
        uuid user_id FK
        uuid tenant_id FK
        enum day_of_week
        time start_time
        time end_time
        int slot_duration_minutes
    }

    Appointment {
        uuid id PK
        uuid tenant_id FK
        uuid patient_id FK
        uuid doctor_id FK
        datetime scheduled_start
        datetime scheduled_end
        enum type
        enum status
        int queue_position
    }

    Visit {
        uuid id PK
        uuid tenant_id FK
        uuid patient_id FK
        uuid doctor_id FK
        uuid appointment_id FK
        text subjective_notes
        text objective_notes
        text assessment
        text plan
        text prescription
    }

    Vitals {
        uuid id PK
        uuid visit_id FK
        uuid tenant_id FK
        float systolic_bp
        float diastolic_bp
        float temperature_f
        float weight_kg
        float height_cm
        int pulse_bpm
    }

    Notification {
        uuid id PK
        uuid tenant_id FK
        uuid appointment_id FK
        enum channel
        enum status
        datetime scheduled_for
        datetime sent_at
    }

    AuditLog {
        uuid id PK
        uuid tenant_id FK
        uuid user_id FK
        string entity_type
        uuid entity_id
        enum action
        jsonb old_values
        jsonb new_values
    }
```

### Key Enums

| Enum | Values |
|------|--------|
| Role | `DOCTOR`, `RECEPTIONIST` (extensible) |
| AppointmentType | `SCHEDULED`, `WALK_IN` |
| AppointmentStatus | `BOOKED`, `CHECKED_IN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`, `NO_SHOW` |
| NotificationChannel | `SMS`, `WHATSAPP` |
| Gender | `MALE`, `FEMALE`, `OTHER`, `PREFER_NOT_TO_SAY` |

---

## Modules

### Auth & Tenant Management

- Tenant signup creates the clinic and first doctor user
- Email/password login with bcrypt, HTTP-only session cookies
- JWT embeds `tenantId` and `role` for every request
- Invite users by email with role assignment
- RBAC via role-permission map (code-based, DB-driven later)

### Patient Management

- Full CRUD with demographics, medical info, allergies, chronic conditions
- Patients tagged to individual users (not roles) via `UserPatient` junction table
- Auto-tag to creating user on registration
- Duplicate detection: same name + DOB + phone triggers a warning
- PostgreSQL full-text search (`tsvector` + GIN index) on name and phone
- Soft-delete only — never hard-delete patient data

### Appointment Scheduling

- Doctors define weekly recurring availability (day, start/end time, slot duration)
- Block specific dates for vacation/holidays
- Slot generation: `availableSlots = allSlots - bookedSlots`
- Scheduled booking with real-time conflict detection
- Walk-in queue: auto-incrementing position, skips slot validation
- Status lifecycle: `BOOKED → CHECKED_IN → IN_PROGRESS → COMPLETED`
- Calendar view (day/week) for doctors, list view for receptionists

### Visits & Vitals

- Visit created from appointment (linked via `appointment_id`)
- SOAP notes: 4 separate fields (Subjective, Objective, Assessment, Plan)
- Vitals: BP, temperature, weight, height, pulse, SpO2
- Prescription text + file attachments (S3/R2)

### Notifications

- SMS + WhatsApp via Twilio
- Auto-scheduled reminders: 24hr and 1hr before appointment
- Cancelled on appointment cancellation
- Fire-and-forget with retry queue (max 3 attempts)

---

## Flowcharts

### Patient Registration

```mermaid
flowchart TD
    A[User clicks 'Add Patient'] --> B[Registration Form]
    B --> C{Valid?}
    C -- No --> D[Show validation errors] --> B
    C -- Yes --> E[Submit to server]
    E --> F{Duplicate detected?<br/>same name + DOB + phone}
    F -- Yes --> G[Warning: patient may already exist]
    G --> H{Create anyway or view existing?}
    H -- View existing --> I[Navigate to existing patient]
    H -- Create anyway --> J[Create patient record]
    F -- No --> J
    J --> K[Auto-tag to creating user]
    K --> L[Write audit log]
    L --> M[Navigate to patient detail]
```

### Appointment Booking

```mermaid
flowchart TD
    A[User starts booking] --> B{Scheduled or Walk-In?}

    B -- Scheduled --> C[Select doctor]
    C --> D[Select date]
    D --> E[Fetch available slots]
    E --> F{Slots available?}
    F -- No --> G[No slots — pick another date] --> D
    F -- Yes --> H[Select time slot]
    H --> I[Select or search patient]
    I --> J[Enter chief complaint]
    J --> K[Submit booking]
    K --> L{Slot still available?}
    L -- Conflict --> M[Slot taken error] --> E
    L -- OK --> N[Create appointment<br/>status: BOOKED]

    B -- Walk-In --> O[Select doctor]
    O --> P[Select or register patient]
    P --> Q[Enter chief complaint]
    Q --> R[Submit walk-in]
    R --> S[Calculate queue position]
    S --> T[Create appointment<br/>status: CHECKED_IN<br/>type: WALK_IN]

    N --> U[Schedule reminders<br/>24hr + 1hr before]
    U --> V[Audit log + confirmation]
    T --> V
```

### User Onboarding

```mermaid
flowchart TD
    A[Doctor visits /signup] --> B[Enter clinic name + credentials]
    B --> C[Create Tenant + User<br/>role: DOCTOR]
    C --> D[Set session cookie]
    D --> E[Redirect to Dashboard]
    E --> F[Set availability schedule]
    F --> G[Navigate to Settings → Users]
    G --> H[Enter email + select role]
    H --> I[Send invite email<br/>with /invite/TOKEN link]

    J[Invited user clicks link] --> K{Token valid?}
    K -- Expired/Invalid --> L[Show error:<br/>link expired, contact admin]
    K -- Valid --> M[Enter name + password]
    M --> N[Create User with assigned role]
    N --> O[Set session cookie]
    O --> P[Redirect to Dashboard]
```

### Appointment Status Lifecycle

```mermaid
stateDiagram-v2
    [*] --> BOOKED : Scheduled appointment
    [*] --> CHECKED_IN : Walk-in arrival

    BOOKED --> CHECKED_IN : Patient arrives
    BOOKED --> CANCELLED : Cancel appointment
    BOOKED --> NO_SHOW : Patient didn't show

    CHECKED_IN --> IN_PROGRESS : Doctor starts consultation
    IN_PROGRESS --> COMPLETED : Doctor finishes + creates visit

    CANCELLED --> [*]
    NO_SHOW --> [*]
    COMPLETED --> [*]
```

---

## RBAC — Permission Matrix

| Permission | Doctor | Receptionist |
|-----------|--------|-------------|
| `patients:create` | Yes | Yes |
| `patients:read` | Yes | Yes |
| `patients:update` | Yes | Yes |
| `appointments:create` | Yes | Yes |
| `appointments:read` | Yes | Yes |
| `appointments:update` | Yes | Yes |
| `appointments:cancel` | Yes | Yes |
| `visits:create` | Yes | No |
| `visits:read` | Yes | No |
| `visits:update` | Yes | No |
| `vitals:create` | Yes | Yes |
| `vitals:read` | Yes | Yes |
| `schedule:manage` | Yes (own) | No |
| `users:invite` | Yes | No |
| `settings:manage` | Yes | No |

---

## Security

- **Tenant isolation**: Prisma middleware on every query — no cross-tenant access
- **Encryption**: AES-256 at rest (database-level), TLS 1.3 in transit
- **Authentication**: bcrypt (cost 12), HTTP-only Secure SameSite cookies, 24hr JWT
- **Authorization**: RBAC checked at tRPC middleware layer
- **Audit logging**: append-only log for every CREATE/UPDATE/DELETE
- **Patient data**: soft-delete only, no hard deletes
- **PHI protection**: no patient data in application logs (console, Sentry)
- **Rate limiting**: 10 attempts per 15 minutes per IP on auth endpoints
- **Invite tokens**: single-use, 72hr expiry, stored hashed

---

## Implementation Phases

### Phase 1 — Core ERP

| Sprint | Weeks | Focus |
|--------|-------|-------|
| 1 | 1-2 | Project setup, Prisma schema, tenant middleware, auth, layout shell |
| 2 | 3-4 | Patient CRUD, search, tagging, list/detail pages, audit logging |
| 3 | 5-6 | Doctor availability, slot generation, booking + walk-in, calendar |
| 4 | 7-8 | Visits, SOAP notes, vitals, Twilio notifications, reminder jobs |
| 5 | 9-10 | User invites, RBAC UI, dashboard, polish, security hardening, deploy |

### Phase 2 — AI Layer (Future Roadmap)

| Feature | Description |
|---------|-------------|
| Smart Scheduling | Suggest optimal appointment times from historical patterns |
| Auto Visit Summaries | LLM summarization of SOAP notes into patient-friendly text |
| No-Show Prediction | ML model scoring no-show probability per appointment |
| Voice-to-Notes | Speech-to-text + LLM structuring into SOAP format |
| Drug Interaction Alerts | Cross-reference prescriptions against interaction database |
| Patient Intake Chatbot | WhatsApp bot collecting info before visit |

**Phase 1 decisions enabling Phase 2:**

- JSONB `attachments` on Visit → store audio files, AI summaries without schema changes
- Separate SOAP fields (not one blob) → LLM extraction and summarization
- Timestamps on all status changes → training data for no-show prediction
- Notification delivery tracking → correlate reminders with no-show rates
- Walk-in `queue_position` → wait time prediction models

---

## Development

```bash
# Install dependencies
pnpm install

# Set up environment
cp .env.example .env

# Push schema to database
pnpm db:push

# Generate Prisma client
pnpm db:generate

# Seed dev data
pnpm db:seed

# Start dev server
pnpm dev
```

### Verification

```bash
pnpm typecheck     # TypeScript strict — must pass with zero errors
pnpm lint          # Biome lint + format check
pnpm test          # Vitest unit/integration tests
pnpm test:e2e      # Playwright E2E tests
```

---

## License

Private — All rights reserved.
