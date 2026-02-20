# Event Sourcing Implementation

## Overview

The Event & State Layer (CORE) is implemented using event sourcing principles. All writes to the system happen by appending events to an append-only event store.

## Architecture

### Event Store
- **Events**: Immutable, append-only records of all state changes
- **ProjectorCheckpoints**: Track processing progress for event projectors

### Event Registry
All event types must be registered in `packages/shared/src/events/registry.ts` with their Zod validation schemas.

## Database Schema

### Event Model
```prisma
model Event {
  id           String    @id @default(uuid())
  practiceId   String
  aggregateType String   @db.VarChar(100)
  aggregateId  String    @db.VarChar(255)
  eventType    String    @db.VarChar(100)
  payloadJson  String    @db.Text
  actorType    ActorType
  actorId      String?   @db.VarChar(255)
  createdAt    DateTime  @default(now())
}
```

### ProjectorCheckpoint Model
```prisma
model ProjectorCheckpoint {
  id           String   @id @default(uuid())
  practiceId   String
  projectorName String @db.VarChar(100)
  lastEventId  String   @db.VarChar(255)
  updatedAt    DateTime @default(now()) @updatedAt
}
```

## API Endpoints

### POST /v1/events/append

Append a new event to the event store.

**Request Body:**
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

**Validation:**
- Requires authentication (`x-user-id` header)
- Requires membership in the practice
- Validates event type is registered
- Validates payload against event type schema

### GET /v1/events/aggregate/:practiceId/:aggregateType/:aggregateId

Get all events for a specific aggregate in chronological order.

**Response:** 200 OK
```json
{
  "events": [
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
  ]
}
```

### GET /v1/events/type/:practiceId/:eventType

Get events by type for a practice.

**Query Parameters:**
- `limit` (optional): Maximum number of events to return (default: 100)

## Event Registry

### Registered Event Types

- `practice.created` - Practice created
- `practice.updated` - Practice updated
- `user.created` - User created
- `membership.created` - Membership created
- `membership.role_changed` - Membership role changed

### Adding New Event Types

1. Define the payload schema in `packages/shared/src/events/schemas.ts`:
```typescript
export const myEventPayloadSchema = z.object({
  field1: z.string(),
  field2: z.number(),
});
```

2. Register it in `packages/shared/src/events/registry.ts`:
```typescript
export const eventRegistry: Record<string, z.ZodSchema> = {
  // ... existing events
  'my.aggregate.event': myEventPayloadSchema,
};
```

3. Rebuild the shared package:
```bash
pnpm build:shared
```

## Append-Only Enforcement

Events are **immutable** and **append-only**:
- No updates: To change state, append a new event
- No deletes: Events are never deleted (only soft-deleted via events if needed)
- All state changes are represented as events

## Multi-Tenancy

All events are scoped by `practiceId`:
- Events can only be appended by users with membership in that practice
- Events can only be queried for practices the user has access to
- Database indexes ensure efficient querying by practice

## Actor Types

- `human`: Actions performed by authenticated users
- `ai`: Actions performed by AI agents
- `system`: System-generated events (e.g., scheduled tasks)

## Testing

Run tests with:
```bash
cd services/api
pnpm test
```

Test coverage includes:
- Schema validation
- Append-only enforcement
- Auth scoping by practiceId
- Event retrieval
- Projector checkpoints

## Migration

After adding the Event and ProjectorCheckpoint models, run:

```bash
pnpm db:migrate
```

This will create the necessary database tables and indexes.
