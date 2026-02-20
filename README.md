# Dental Growth OS

Production-ready monorepo for dental practice management platform.

## Architecture

```
UI → Event/State Core → Template Engine → Rules/Safety → Orchestrator → Agents → Execution Tools → External boundary
```

### Event Sourcing (CORE)

The system uses **event sourcing** for the Event & State Layer:
- **Append-only event store**: All state changes are recorded as immutable events
- **Event Registry**: Type-safe event schemas with Zod validation
- **Multi-tenant**: All events scoped by `practiceId`
- **Projectors**: Background workers process events to build read models

## Tech Stack

- **Frontend**: Next.js 14 (Dashboard + Landing) with Tailwind CSS
- **Backend**: Fastify API service
- **Worker**: Node.js worker for projectors and automation
- **Database**: PostgreSQL with Prisma ORM
- **Monorepo**: pnpm workspaces + Turbo
- **Auth**: NextAuth.js (Credentials provider)

## Project Structure

```
.
├── apps/
│   ├── dashboard/      # Next.js dashboard app (port 3000)
│   └── landing/        # Next.js landing page (port 3002)
├── services/
│   ├── api/            # Fastify API service (port 3001)
│   └── worker/         # Worker service for projectors/automation
├── packages/
│   └── shared/         # Shared Zod schemas, TypeScript types, and Event Registry
├── docker-compose.yml  # PostgreSQL database
└── package.json        # Root workspace configuration
```

## Prerequisites

- Node.js >= 18.0.0
- pnpm >= 8.0.0
- Docker and Docker Compose

## Setup Instructions

### 1. Install Dependencies

```bash
pnpm install
```

### 2. Start PostgreSQL Database

```bash
docker-compose up -d
```

Verify the database is running:
```bash
docker-compose ps
```

### 3. Configure Environment Variables

Copy the example environment files and fill in the values:

```bash
# API Service
cp services/api/.env.example services/api/.env

# Dashboard App
cp apps/dashboard/.env.example apps/dashboard/.env

# Worker Service (optional)
cp services/worker/.env.example services/worker/.env
```

**Required environment variables:**

**services/api/.env:**
```env
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/dental_growth_os?schema=public
JWT_SECRET=your-super-secret-jwt-key-minimum-32-characters-long
API_URL=http://localhost:3001
```

**apps/dashboard/.env:**
```env
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-nextauth-secret-key-change-in-production
NEXT_PUBLIC_API_URL=http://localhost:3001
```

### 4. Build Shared Package

```bash
pnpm build:shared
```

Builds Event & State Layer schemas and types used by the API and dashboard.

### 5. Generate Prisma Client

```bash
pnpm db:generate
```

### 6. Run Database Migrations

```bash
pnpm db:migrate
```

This will create the database schema with:
- `practices` table
- `users` table
- `memberships` table (multi-tenant relationships)
- `patients` table (non-clinical patient data)
- `events` table (event store - append-only)
- `projector_checkpoints` table (projector state tracking)
- `flow_templates` table (prebuilt flow templates)
- `installed_templates` table (practice-installed templates)
- `practice_configs` table (rules & safety configuration)

### 7. Start Development Servers

```bash
pnpm dev
```

This will start all services concurrently:
- **Dashboard**: http://localhost:3000 (Main application; login required in production build)
- **Landing**: http://localhost:3002 (Public marketing page)
- **API**: http://localhost:3001 (Backend API)
- **Worker**: Running in background (Template processing)

**Important**: The dashboard runs on port **3000**, not 3002. The landing page (port 3002) has a link to redirect to the dashboard.

### Production build and login

When you run a full build and start the dashboard in production (`pnpm build` then `next start` in `apps/dashboard`), the **login system is enforced**:

- All routes except `/login` and `/signup` require authentication.
- Unauthenticated users are redirected to `/login`. After signing in, they are sent back to the page they tried to open (`callbackUrl`).
- Set `NEXTAUTH_URL` and `NEXTAUTH_SECRET` in your production environment (see `apps/dashboard/.env.example`).

## Available Scripts

### Root Level

- `pnpm dev` - Start all services in development mode
- `pnpm build` - Build all packages and apps
- `pnpm test` - Run tests across all packages
- `pnpm lint` - Lint all packages
- `pnpm format` - Format code with Prettier
- `pnpm db:migrate` - Run database migrations
- `pnpm db:studio` - Open Prisma Studio (database GUI)
- `pnpm db:generate` - Generate Prisma Client

### Individual Packages

Each package/app has its own scripts. Navigate to the directory and run:
- `pnpm dev` - Start development server
- `pnpm build` - Build for production
- `pnpm start` - Start production server
- `pnpm lint` - Lint code
- `pnpm test` - Run tests

## Multi-Tenant Architecture

The system is built with multi-tenancy in mind:

- **Practice**: Top-level tenant entity
- **User**: Can belong to multiple practices
- **Membership**: Links users to practices with roles:
  - `owner` - Full control
  - `admin` - Administrative access
  - `frontdesk` - Front desk operations
  - `viewer` - Read-only access

Every table includes `practiceId` for tenant isolation.

## Event Sourcing

### Core Principles

- **Append-only**: Events are immutable and cannot be updated or deleted
- **Event Registry**: All event types must be registered with Zod schemas
- **State Reconstruction**: Current state is derived by replaying events
- **Audit Trail**: Complete history of all state changes

### Database Schema

**Events Table:**
- `id` - Unique event identifier
- `practiceId` - Tenant scope
- `aggregateType` - Type of aggregate (e.g., "practice", "user")
- `aggregateId` - ID of the aggregate instance
- `eventType` - Type of event (e.g., "practice.created")
- `payloadJson` - Event payload as JSON
- `actorType` - Who performed the action: `human`, `ai`, or `system`
- `actorId` - ID of the actor
- `createdAt` - Timestamp

**ProjectorCheckpoints Table:**
- Tracks processing progress for event projectors
- Enables resumable event processing

### Event API Endpoints

#### POST /v1/events/append

Append a new event to the event store.

**Request:**
```json
{
  "practiceId": "uuid",
  "aggregateType": "practice",
  "aggregateId": "uuid",
  "eventType": "practice.created",
  "payload": { "name": "Practice Name" },
  "actorType": "human",
  "actorId": "uuid"
}
```

**Response:** 201 Created
```json
{
  "id": "uuid",
  "practiceId": "uuid",
  "aggregateType": "practice",
  "aggregateId": "uuid",
  "eventType": "practice.created",
  "payload": { "name": "Practice Name" },
  "actorType": "human",
  "actorId": "uuid",
  "createdAt": "2024-01-01T00:00:00Z"
}
```

#### GET /v1/events/aggregate/:practiceId/:aggregateType/:aggregateId

Get all events for a specific aggregate in chronological order.

#### GET /v1/events/type/:practiceId/:eventType

Get events by type for a practice.

### Registered Event Types

**Practice Events:**
- `practice.created` - Practice created
- `practice.updated` - Practice updated

**User Events:**
- `user.created` - User created

**Membership Events:**
- `membership.created` - Membership created
- `membership.role_changed` - Membership role changed

**Appointment Events:**
- `appointment.requested` - New appointment requested
- `appointment.status_changed` - Appointment status changed

**Template Engine Events:**
- `message.queued` - SMS/Email message queued for sending
- `task.created` - Task created for human review
- `ai_action.proposed` - AI action proposed (requires review)

### Adding New Event Types

1. Define payload schema in `packages/shared/src/events/schemas.ts`
2. Register in `packages/shared/src/events/registry.ts`
3. Rebuild shared package: `pnpm build:shared`

**Event & State Layer (CORE)** includes:
- **Patient** (non-clinical data only): `patient.created`, `patient.updated`
- **Appointment** (type, duration, status) and lifecycle state
- **Immutable events**, **append-only audit logs**
- **Example events**: `booking_requested`, `booking_confirmed`, `consult_completed`, `treatment_completed`, `recall_due`, `no_show`

See [EVENT_SOURCING.md](./EVENT_SOURCING.md) and [docs/EVENT_STATE_LAYER.md](./docs/EVENT_STATE_LAYER.md) for details.

## Prebuilt Flow Template Engine

The Prebuilt Flow Template Engine runs workflows when events occur. Templates are **versioned** and **installed automatically on onboarding** (when a practice is created).

### Features

- **Event-driven**: Templates trigger on registered event types (e.g. `appointment.status_changed`, `booking_confirmed`, `consult_completed`, `recall_due`, `no_show`)
- **Step types**: SMS, Email, Task creation, AI action proposals
- **Multi-tenant**: Global templates; each practice has its own installed set and active version

### Flow categories (installed on onboarding)

| Flow | Description |
|------|-------------|
| **Confirmation flows** | SMS when appointment is confirmed |
| **Reminder flows** | Email reminders before appointments |
| **Consult follow-up** | Follow-up after consultation appointments |
| **Treatment follow-up** | Follow-up after treatment appointments |
| **Review / Referral** | Review or referral requests after completion |
| **Recall / Reactivation** | Recall and reactivation (e.g. AI-proposed actions) |

No-Show Recovery is also installed (follow-up and task for no-show patients).

### Step Types

**send_sms:**
```json
{
  "type": "send_sms",
  "templateKey": "appointment_confirmed",
  "variablesJson": { "patientName": "{{patientName}}" }
}
```

**send_email:**
```json
{
  "type": "send_email",
  "templateKey": "appointment_reminder",
  "variablesJson": { "patientName": "{{patientName}}" }
}
```

**create_task:**
```json
{
  "type": "create_task",
  "title": "Follow up with {{patientName}}",
  "dueOffsetMinutes": 60,
  "assignedRole": "frontdesk"
}
```

**propose_ai_action:**
```json
{
  "type": "propose_ai_action",
  "agentName": "recall_coordinator",
  "contextHintsJson": { "patientName": "{{patientName}}" }
}
```

### API Endpoints

- `GET /v1/templates?practiceId=xxx` - List installed templates

## Execution Tools

Available tools: **send_sms**, **send_email**, **place_call** (voice AI optional), **create_task**, **create_booking_request**, **update_lifecycle_state**, **escalate_to_front_desk**, plus **propose_ai_action** and **auto_confirm** for AI and booking. See [docs/EXECUTION_TOOLS.md](./docs/EXECUTION_TOOLS.md).

## AI Agent Layer

Agents have **strict permissions**: each agent may only use the tools it is allowed to use. Registered agents: **Intake**, **Booking**, **Confirmation**, **Follow-up**, **Review**, **Referral**, **Recall**, **Reactivation**, **No-show Recovery**, **QA**, **Optimizer**. When a template or system proposes an AI action (`propose_ai_action`), the agent name must be registered and the agent must be permitted to use that tool. See [docs/AI_AGENT_LAYER.md](./docs/AI_AGENT_LAYER.md).

## Rules & Safety Layer

The Rules & Safety Layer evaluates all actions **before** execution to ensure compliance and safety.

### Safety Rules

1. **Consent Check**
   - SMS only sent if `consentSms = true`
   - Email only sent if `consentEmail = true`
   - **Decision**: `deny` if consent not given

2. **Quiet Hours**
   - Blocks outbound messages during configured quiet hours
   - Configurable per practice (e.g., 22:00 - 08:00)
   - Supports hours spanning midnight
   - **Decision**: `deny` during quiet hours

3. **Frequency Cap**
   - Limits messages per patient per day
   - Default: 5 messages/day (configurable)
   - **Decision**: `deny` if limit exceeded

4. **Calendar Rule - Booking Mode**
   - `MANUAL_APPROVAL` mode: Blocks auto-confirm, requires human review
   - `DIRECT_PMS` mode: Allows auto-confirm
   - **Decision**: `require_human` in MANUAL_APPROVAL mode

5. **Sentiment Gate**
   - If patient flagged `angry = true`: Requires human review
   - Escalates to frontdesk role
   - **Decision**: `require_human` with escalation

6. **Non-clinical Filter**
   - Blocks AI actions containing clinical advice keywords
   - Keywords: diagnosis, treatment, prescription, medication, symptom, etc.
   - **Decision**: `require_human` for clinical content

### Evaluation Result

```typescript
{
  decision: 'allow' | 'deny' | 'require_human',
  reason?: string,
  escalationRole?: 'frontdesk' | 'admin' | 'owner'
}
```

### Practice Configuration

Configure rules via Settings page or API:

- `quietHoursStart` - Start time (HH:mm format)
- `quietHoursEnd` - End time (HH:mm format)
- `maxMessagesPerDay` - Maximum messages per patient per day (1-100)
- `bookingMode` - `DIRECT_PMS` or `MANUAL_APPROVAL`

### API Endpoints

- `GET /v1/settings/config?practiceId=xxx` - Get practice configuration
- `PUT /v1/settings/config?practiceId=xxx` - Update configuration (admin/owner only)

### Dashboard

- **Settings Page** (`/settings`): Configure all safety rules
  - Quiet hours configuration
  - Message frequency limits
  - Booking mode selection

## Appointment Lifecycle

Appointments follow a strict state machine with validated transitions.

### Status Flow

```
requested → penciled → confirmed → completed
                              ↓
                          no_show
                              ↓
                        rescheduled
```

**Special transitions:**
- `any → canceled` (human only)
- `confirmed → no_show`
- `confirmed → rescheduled` (requires new scheduledFor)
- `requested/penciled → needs_manual_review`

### Transition Rules

- **Standard flow**: `requested → penciled → confirmed → completed`
- **Alternative flows**: `confirmed → no_show` or `confirmed → rescheduled`
- **Cancel**: Any status → `canceled` (requires human actor)
- **Manual review**: Can be flagged for review at any point

### API Endpoints

- `POST /v1/appointments/request?practiceId=xxx` - Request new appointment
- `POST /v1/appointments/:id/status?practiceId=xxx` - Change status (validates transitions)
- `POST /v1/appointments/:id/reschedule?practiceId=xxx` - Reschedule appointment
- `GET /v1/appointments/:id?practiceId=xxx` - Get appointment details
- `GET /v1/appointments?practiceId=xxx` - List appointments (with filters)

### Dashboard

- **Appointments Page** (`/appointments`): List all appointments
- **Appointment Detail** (`/appointments/:id`): View and manage appointment
  - Status change buttons (only valid transitions shown)
  - Reschedule form (for confirmed appointments)
  - Clear validation error display

## Authentication Flow

1. User registers/logs in via Dashboard (NextAuth)
2. NextAuth validates credentials against API
3. API returns user with memberships
4. Session stored in JWT (NextAuth)
5. API requests include `x-user-id` header
6. API validates membership and role for each request

## Database Management

### Prisma Studio

Visual database browser:
```bash
pnpm db:studio
```

### Create Migration

```bash
cd services/api
pnpm prisma migrate dev --name migration_name
```

### Reset Database

```bash
cd services/api
pnpm prisma migrate reset
```

## Development Workflow

1. **Start database**: `docker-compose up -d`
2. **Install dependencies**: `pnpm install`
3. **Setup environment**: Copy `.env.example` files
4. **Generate Prisma client**: `pnpm db:generate`
5. **Run migrations**: `pnpm db:migrate`
6. **Start dev servers**: `pnpm dev`

## Production Deployment

1. Build all packages: `pnpm build`
2. Set production environment variables
3. Run migrations: `pnpm db:migrate`
4. Start services:
   - API: `cd services/api && pnpm start`
   - Dashboard: `cd apps/dashboard && pnpm start`
   - Landing: `cd apps/landing && pnpm start`
   - Worker: `cd services/worker && pnpm start`

## Code Quality

- **ESLint**: Configured for TypeScript
- **Prettier**: Code formatting
- **TypeScript**: Strict mode enabled
- **Zod**: Runtime validation for API schemas and event payloads
- **Vitest**: Unit and integration tests

## Constraints

- **Non-clinical data only**: No patient health records
- **No insurance verification**: Insurance features not included
- **No payments/collections**: Payment processing not included

## Troubleshooting

### Database Connection Issues

1. Verify PostgreSQL is running: `docker-compose ps`
2. Check DATABASE_URL in `services/api/.env`
3. Ensure database exists: `docker-compose exec postgres psql -U postgres -l`

### Port Already in Use

If ports 3000, 3001, or 3002 are in use:
- Dashboard: Change port in `apps/dashboard/package.json` dev script
- API: Change PORT in `services/api/.env`
- Landing: Change port in `apps/landing/package.json` dev script

### Prisma Client Not Generated

Run: `pnpm db:generate`

### Module Resolution Issues

Ensure shared package is built:
```bash
pnpm build:shared
```

### Event Validation Errors

If you see "Event type not registered" errors:
1. Check `packages/shared/src/events/registry.ts` for registered event types
2. Rebuild shared package: `pnpm build:shared`
3. Restart API server

### Running Tests

```bash
# Run all tests
pnpm test

# Run API tests specifically
cd services/api
pnpm test

# Run tests in watch mode
cd services/api
pnpm test --watch

# Run specific test suites
cd services/api
pnpm test rules.test.ts      # Rules & Safety tests
pnpm test appointments.test.ts # Appointment lifecycle tests
pnpm test templates.test.ts    # Template engine tests
pnpm test events.test.ts       # Event sourcing tests
```

### Rules Evaluation Errors

If actions are being denied unexpectedly:
1. Check practice configuration: `GET /v1/settings/config?practiceId=xxx`
2. Verify patient consent flags (if implemented)
3. Check quiet hours configuration
4. Review message frequency limits
5. Check booking mode settings

## Key Features

### Event Sourcing
- Complete audit trail of all state changes
- Append-only event store
- State reconstruction by replaying events

### Template Engine
- Automatic workflow execution on events
- Pre-built templates for common scenarios
- Customizable step types (SMS, Email, Tasks, AI Actions)

### Rules & Safety
- Pre-execution validation of all actions
- Configurable safety rules per practice
- Human escalation for sensitive actions

### Appointment Management
- State machine with validated transitions
- Multi-status support (requested, penciled, confirmed, completed, etc.)
- Rescheduling and cancellation support

## Additional Documentation

- [docs/FOUNDATION.md](./docs/FOUNDATION.md) - Foundation checklist (stack, folder structure, multi-tenant, docker-compose, Prisma, dev script, health route, README)
- [EVENT_SOURCING.md](./EVENT_SOURCING.md) - Detailed event sourcing documentation
- [docs/EVENT_STATE_LAYER.md](./docs/EVENT_STATE_LAYER.md) - Event & State Layer (CORE): Patient, Appointment, lifecycle events, audit logs
- [docs/TEMPLATE_ENGINE.md](./docs/TEMPLATE_ENGINE.md) - Prebuilt Flow Template Engine: confirmation, reminder, follow-up, recall (versioned, installed on onboarding)
- [docs/AI_AGENT_LAYER.md](./docs/AI_AGENT_LAYER.md) - AI Agent Layer: Intake, Booking, Confirmation, Follow-up, Review, Referral, Recall, Reactivation, No-show Recovery, QA, Optimizer (strict permissions per agent)
- [docs/EXECUTION_TOOLS.md](./docs/EXECUTION_TOOLS.md) - Execution tools: send_sms, send_email, place_call, create_task, create_booking_request, update_lifecycle_state, escalate_to_front_desk
- [docs/BOOKING_FLOW.md](./docs/BOOKING_FLOW.md) - Booking flow: Patient requests time → Booking Agent (Shadow Calendar) → PENCILED → Front desk notified → CONFIRM/ADJUST → System sends confirmation
- [SETUP.md](./SETUP.md) - Quick setup guide and troubleshooting

## License

Private - All rights reserved
