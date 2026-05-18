# Architecture: Data Model

> **Source of truth for spec writers.** When filling the *Data Model Impact* section of a `spec-requirements.md`, reference this file to identify affected entities, field constraints, business rules, and DB-level constraints that must be reflected in FRs and the Definition of Done.

---

## Entity Relationship Overview

```
users ───────────────────────────────────────┐
  │                                          │
  │ technician_id (nullable FK)              │ user_id (acting user FK)
  ▼                                          ▼
service_orders ──────────────── order_logs
  │
  │ customer_id (FK)
  ▼
customers
```

| Relationship | Cardinality | Notes |
|---|---|---|
| `customers` → `service_orders` | 1 : N | A customer can have many orders; an order belongs to exactly one customer |
| `users` → `service_orders` (technician) | 1 : N (nullable) | An order may have no assigned technician at creation |
| `service_orders` → `order_logs` | 1 : N | Every status change produces exactly one log entry |
| `users` → `order_logs` (acting user) | 1 : N | Every log records who performed the action |

---

## Entities

### `customers`

Stores contact information of people who bring devices for service.

| Column | Type | Required | Constraints | Notes |
|--------|------|----------|-------------|-------|
| `id` | `UUID` | Yes | PK, `@default(uuid())` | Never auto-increment |
| `name` | `TEXT` | Yes | 2–100 chars | Full name; no format restriction beyond length |
| `phone` | `TEXT` | Yes | UNIQUE, digits only, 7–15 chars | Primary business identifier; regex `^\d{7,15}$` |
| `email` | `TEXT` | No | Valid email format if present | Nullable; required to receive status notifications |
| `created_at` | `TIMESTAMPTZ` | Yes | `DEFAULT now()` | Set by DB on insert — never by application code |
| `archived_at` | `TIMESTAMPTZ` | No | Nullable | Soft-delete timestamp; `NULL` = active customer |

**Business rules:**
- `phone` is the unique identifier for a customer across the entire system
- A customer cannot be hard-deleted if they have any `service_orders` (open or closed) — use soft delete (`archived_at = now()`)
- Archived customers (`archived_at IS NOT NULL`) must not appear in search results unless explicitly requested
- `email` is optional at registration but required by the notification feature (enforce at feature level, not DB level)

**Zod validation reference:**
```typescript
z.object({
  name:  z.string().min(2).max(100),
  phone: z.string().regex(/^\d{7,15}$/, 'Phone must be 7–15 digits'),
  email: z.string().email().optional(),
})
```

---

### `service_orders`

Tracks a device repair or maintenance job from intake to delivery.

| Column | Type | Required | Constraints | Notes |
|--------|------|----------|-------------|-------|
| `id` | `UUID` | Yes | PK, `@default(uuid())` | |
| `customer_id` | `UUID` | Yes | FK → `customers.id`, NOT NULL | Cannot be null; order must belong to a registered customer |
| `technician_id` | `UUID` | No | FK → `users.id`, nullable | Assigned after creation; required before `IN_PROGRESS` |
| `status` | `ENUM` | Yes | See `OrderStatus` | Default: `DRAFT` on creation |
| `device_brand` | `TEXT` | Yes | Non-empty, max 100 chars | e.g., "Apple", "Samsung" |
| `device_model` | `TEXT` | Yes | Non-empty, max 100 chars | e.g., "iPhone 14 Pro" |
| `serial_number` | `TEXT` | No | Nullable, max 100 chars | May not be available at intake |
| `reported_issue` | `TEXT` | Yes | Non-empty, max 2000 chars | Problem description as reported by the customer |
| `estimated_price` | `DECIMAL(10,2)` | Yes | >= 0 | Quote given to the customer at intake |
| `final_price` | `DECIMAL(10,2)` | No | Nullable, >= 0 | Set when order reaches `DONE`; may differ from estimate |
| `estimated_date` | `DATE` | No | Nullable | Promised completion date |
| `created_at` | `TIMESTAMPTZ` | Yes | `DEFAULT now()` | Set by DB on insert |
| `updated_at` | `TIMESTAMPTZ` | Yes | `@updatedAt` | Managed by Prisma on every write |

**Business rules:**
- `status` transitions must follow the state machine defined below — the service layer validates every transition
- `final_price` must be set before an order can transition to `DONE`
- `technician_id` must be set before an order can transition to `IN_PROGRESS`
- Orders are never hard-deleted — the terminal states are `DELIVERED` and `CANCELLED`
- Monetary fields use `DECIMAL(10,2)` — never `FLOAT` or `DOUBLE`

**Zod validation reference (order creation):**
```typescript
z.object({
  customerId:     z.string().uuid(),
  deviceBrand:    z.string().min(1).max(100),
  deviceModel:    z.string().min(1).max(100),
  serialNumber:   z.string().max(100).optional(),
  reportedIssue:  z.string().min(1).max(2000),
  estimatedPrice: z.number().nonnegative(),
  estimatedDate:  z.string().date().optional(),
})
```

---

### `order_logs`

Immutable audit trail of every status transition on a service order.

| Column | Type | Required | Constraints | Notes |
|--------|------|----------|-------------|-------|
| `id` | `UUID` | Yes | PK, `@default(uuid())` | |
| `order_id` | `UUID` | Yes | FK → `service_orders.id` | |
| `user_id` | `UUID` | Yes | FK → `users.id` | The authenticated user who triggered the transition |
| `from_status` | `ENUM` | No | Nullable | `NULL` on order creation — there is no prior status |
| `to_status` | `ENUM` | Yes | NOT NULL | The resulting status after the transition |
| `comment` | `TEXT` | No | Nullable, max 1000 chars | Optional technician or manager note |
| `created_at` | `TIMESTAMPTZ` | Yes | `DEFAULT now()` | |

**Business rules:**
- Log entries are **immutable** — never updated or deleted; no `updated_at`
- Every status transition in `service_orders` must produce exactly one `order_logs` entry **in the same DB transaction**
- Order creation produces a log entry: `from_status = NULL`, `to_status = DRAFT`
- `user_id` must always be the authenticated user from the JWT — never accept it from the request body

---

### `users`

System users with role-based access. Managed by administrators; not customer-facing.

| Column | Type | Required | Constraints | Notes |
|--------|------|----------|-------------|-------|
| `id` | `UUID` | Yes | PK, `@default(uuid())` | |
| `name` | `TEXT` | Yes | Non-empty, max 100 chars | Display name |
| `email` | `TEXT` | Yes | UNIQUE, valid email | Used as login identifier |
| `role` | `ENUM` | Yes | See `UserRole` | Determines endpoint access via `requireRole` middleware |
| `password_hash` | `TEXT` | Yes | bcrypt hash | **Never** returned in API responses or logs |
| `created_at` | `TIMESTAMPTZ` | Yes | `DEFAULT now()` | |

**Business rules:**
- `password_hash` is computed with bcrypt at cost factor >= 10
- `password_hash` must never appear in any API response, log output, or error message
- `email` is the login identifier — unique across all users
- The `role` field gates access via `requireRole` middleware in every protected route

**Zod validation reference (login input):**
```typescript
z.object({
  email:    z.string().email(),
  password: z.string().min(1),
})
```

---

## Enums

### `OrderStatus`

| Value | Description | Who sets it | Via action |
|-------|-------------|-------------|------------|
| `DRAFT` | Created, not yet accepted | System | `POST /service-orders` |
| `PENDING` | Accepted, waiting for a technician | RECEPTIONIST, MANAGER | `accept()` |
| `IN_PROGRESS` | Technician actively working | TECHNICIAN, MANAGER | `start()` |
| `WAITING_PARTS` | Work paused — waiting for spare parts | TECHNICIAN, MANAGER | `waitParts()` |
| `DONE` | Repair complete, awaiting pickup | TECHNICIAN, MANAGER | `complete()` |
| `DELIVERED` | Device returned to customer | RECEPTIONIST, MANAGER | `deliver()` |
| `CANCELLED` | Order abandoned — **terminal** | RECEPTIONIST, MANAGER | `cancel()` |

### `UserRole`

| Value | Description | Key capabilities |
|-------|-------------|-----------------|
| `RECEPTIONIST` | Front desk — intake, customer comms | Register customers, open orders, mark as delivered, search |
| `TECHNICIAN` | Workshop — device repair | Update order status, log work progress |
| `MANAGER` | Operations — oversight | All actions + reports, pricing, technician workload |

---

## State Machine — `service_orders.status`

```
          ┌─────────┐
          │  DRAFT  │  ← set by system on POST /service-orders
          └────┬────┘
               │ accept()  [RECEPTIONIST, MANAGER]
          ┌────▼────┐
          │ PENDING │◄────────────────────────────┐
          └────┬────┘                              │
               │ start()  [TECHNICIAN, MANAGER]    │
       ┌───────▼────────┐                          │
       │  IN_PROGRESS   │── waitParts() ──► WAITING_PARTS
       └───────┬────────┘   [TECH, MGR]  ◄──────────┘
               │                          resume() [TECH, MGR]
               │ complete()  [TECHNICIAN, MANAGER]
          ┌────▼────┐
          │  DONE   │
          └────┬────┘
               │ deliver()  [RECEPTIONIST, MANAGER]
        ┌──────▼──────┐
        │  DELIVERED  │  ← terminal — no further transitions
        └─────────────┘

Any status except DELIVERED and CANCELLED → CANCELLED  (RECEPTIONIST, MANAGER)
```

**Valid transitions (service layer must validate these):**

| From | To | Action | Allowed roles |
|---|---|---|---|
| `DRAFT` | `PENDING` | `accept()` | RECEPTIONIST, MANAGER |
| `PENDING` | `IN_PROGRESS` | `start()` | TECHNICIAN, MANAGER |
| `IN_PROGRESS` | `WAITING_PARTS` | `waitParts()` | TECHNICIAN, MANAGER |
| `WAITING_PARTS` | `IN_PROGRESS` | `resume()` | TECHNICIAN, MANAGER |
| `IN_PROGRESS` | `DONE` | `complete()` | TECHNICIAN, MANAGER |
| `DONE` | `DELIVERED` | `deliver()` | RECEPTIONIST, MANAGER |
| `DRAFT`–`DONE` | `CANCELLED` | `cancel()` | RECEPTIONIST, MANAGER |
| `DELIVERED` | *(any)* | — | Invalid — terminal state |
| `CANCELLED` | *(any)* | — | Invalid — terminal state |

> Any attempt to perform an unlisted transition must return `HTTP 409` with `error.code = "INVALID_STATUS_TRANSITION"`.

---

## Cross-Cutting Patterns

### UUID Primary Keys
All entities use UUID v4. Never use auto-increment integers. UUIDs received as input must be validated with `z.string().uuid()`.

### Audit Fields
| Pattern | Entities | Rule |
|---|---|---|
| `created_at DEFAULT now()` | All | Set by DB — never by application code |
| `updated_at @updatedAt` | `service_orders` | Managed automatically by Prisma |
| Acting `user_id` on writes | `order_logs` | Every order mutation records who performed it |

### Soft Delete
Only `customers` uses soft delete (`archived_at`). All other entities use terminal state transitions or are immutable.
- "Delete customer" = `UPDATE customers SET archived_at = now()` — never `DELETE`
- All customer queries must include `WHERE archived_at IS NULL` unless explicitly fetching archived records

### Monetary Values
`estimated_price` and `final_price` are `DECIMAL(10,2)`. Never `FLOAT` or `DOUBLE`. In TypeScript, represent as `number` and validate with `z.number().nonnegative()`. When formatting for display, always use 2 decimal places.

### Timestamps
All timestamps are stored in UTC as `TIMESTAMPTZ`. Always return them in ISO 8601 format (`2024-01-15T10:30:00.000Z`). Never store or return local time.

### `user_id` on protected writes
The acting user's `id` must always come from the decoded JWT (`req.user.id`) — never from the request body. Routes must reject any attempt to pass `userId` in the body for audit-sensitive operations.

---

## Spec Writing Guide — Data Model Impact Section

When filling the *Data Model Impact* section of a `spec-requirements.md`, answer these five questions:

**1. Does this story create a new entity?**
- New Prisma model → migration required
- Define all columns, types, nullability, and constraints in the spec's FR table
- Add the new entity to this file after implementation

**2. Does this story add or modify fields on an existing entity?**
- Specify column name (`snake_case`), type, nullability, and any new constraints
- Adding a `NOT NULL` column to a table with existing rows → migration strategy required in the spec (default value or data backfill)
- Adding a `UNIQUE` constraint → spec must address how existing duplicates are handled

**3. Does this story enforce a new business rule at the DB level?**
- New uniqueness rule → `@unique` or `@@unique` in Prisma schema + migration
- New FK relationship → `@relation` in Prisma schema
- New enum value → update the enum table above and the Prisma schema

**4. Does this story involve a status transition?**
- Add the new transition to the valid transition table above
- Specify which roles may trigger it
- The service layer validates every transition — never trust the client's `status` value
- Each transition generates one `order_logs` entry in the same DB transaction

**5. Does this story produce audit data?**
- Status changes on `service_orders` → `order_logs` entry is mandatory, not optional
- Any write traceable to a user → record `user_id` from the JWT, not from the request body

**Ready-to-use template for spec-requirements.md:**

```markdown
## Data Model Impact

**Entities affected:** `[table_name]`, `[table_name]`

| Entity | Change | Details |
|--------|--------|---------|
| `customers` | Modify | Add column `archived_at TIMESTAMPTZ NULL` (soft delete) |
| `service_orders` | New | Full schema — see FR list |

Migration required: yes / no
Migration strategy for existing data: [default value / backfill / N/A]
```
