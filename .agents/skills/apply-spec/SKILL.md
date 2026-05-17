# Skill: Apply Spec (apply-spec)

## Description

Implements in code everything defined in a `spec-requirements.md` + `spec-bdd.md` pair for a given JIRA ticket. Produces all four application layers (model → repository → service → route), the Prisma migration, and the full test suite (unit, integration, e2e) — one test per BDD scenario.

## Trigger

Use this skill when the user:
- Asks to implement a ticket from its spec
- Says "implement JIRA-123", "apply the spec", "code this story", "build this from the spec"
- Has both `spec-requirements.md` and `spec-bdd.md` ready and wants the code generated

## Pre-conditions

Before running this skill, both spec files must exist:
- `specs/<TICKET>/spec-requirements.md`
- `specs/<TICKET>/spec-bdd.md`

If either is missing, stop and tell the user which skill to run first (`enrich-us` or `bdd-creator`).

---

## Instructions

You are a senior full-stack engineer on the TechShop project. The spec is the contract — implement exactly what it says, nothing more. Do not add fields, endpoints, or behaviors not described in the spec. Do not skip anything marked **Must**.

---

### Step 1 — Read and Parse the Spec

Read both files in full:

1. `specs/<TICKET>/spec-requirements.md` — extract:
   - Entity name and HTTP resource path (e.g., `customers`, `POST /customers`)
   - Field list with types, constraints, and required/optional status
   - Business rules (uniqueness, state transitions, domain invariants)
   - Allowed roles (for auth middleware)
   - Non-functional requirements (performance, security, accessibility)
   - Out-of-scope items (do NOT implement these)

2. `specs/<TICKET>/spec-bdd.md` — extract:
   - Every scenario with its Given/When/Then steps
   - Expected HTTP status codes and response body structure
   - Error codes (`DUPLICATE_PHONE`, `VALIDATION_ERROR`, `UNAUTHORIZED`, etc.)
   - Test file path and `describe` block name for each scenario

Build a mental implementation checklist before writing any code.

---

### Step 2 — Derive the Implementation Map

Silently derive the following before writing code:

| Artifact | Derived from |
|---|---|
| Zod schema fields + rules | FR field list + format constraints |
| Prisma model fields + DB constraints | FR field list + uniqueness rules |
| Service function signatures | BDD Scenario When/Then steps |
| Repository method names | Service calls to the DB |
| Route method + path + middleware | Spec HTTP method, path, allowed roles |
| Error codes and HTTP status map | Scenario Then assertions |
| Test `describe` block names | `spec-bdd.md` test file annotations |

---

### Step 3 — Implement the Four Layers (bottom-up)

Implement in this exact order. Each layer depends on the one below it.

---

#### Layer 1 — Model (`src/models/<resource>.model.ts`)

Define TypeScript types and Zod validation schemas.

```typescript
// src/models/customer.model.ts
import { z } from 'zod'

export const CreateCustomerSchema = z.object({
  name: z.string().min(2).max(100),
  phone: z.string().regex(/^\d{7,15}$/, 'Phone must be 7–15 digits'),
  email: z.string().email().optional(),
})

export type CreateCustomerInput = z.infer<typeof CreateCustomerSchema>

export type Customer = {
  id: string
  name: string
  phone: string
  email: string | null
  createdAt: Date
}
```

Rules:
- Zod schema names: `<Verb><Resource>Schema` (e.g., `CreateCustomerSchema`, `UpdateOrderSchema`)
- TypeScript input type: `<Verb><Resource>Input`
- TypeScript entity type: matches the DB model — never expose `password_hash` or internal fields
- Validation messages must be human-readable (they appear in API error responses)
- The Zod schema is the single source of truth for validation — do not duplicate field checks in the service

---

#### Layer 2 — Repository (`src/repositories/<resource>.repository.ts`)

Data access only. No business logic. No HTTP concerns. Returns domain types or `null`.

```typescript
// src/repositories/customer.repository.ts
import { prisma } from '../utils/prisma'
import type { CreateCustomerInput, Customer } from '../models/customer.model'

export async function findCustomerByPhone(phone: string): Promise<Customer | null> {
  return prisma.customer.findUnique({ where: { phone } })
}

export async function createCustomer(data: CreateCustomerInput): Promise<Customer> {
  return prisma.customer.create({ data })
}
```

Rules:
- One file per resource — not one file per operation
- Function names use verbs: `findBy*`, `create*`, `update*`, `archive*`
- Never throw business errors — return `null` for not-found; let the service layer interpret it
- Select only the fields needed; never return `password_hash` from a user query
- Catch Prisma error `P2002` (unique constraint) only if you need to normalize it into a domain error — re-throw as `DomainError` so the service handles it

---

#### Layer 3 — Service (`src/services/<resource>.service.ts`)

Business logic only. No HTTP context (`req`, `res`). Throws typed domain errors.

```typescript
// src/services/customer.service.ts
import { findCustomerByPhone, createCustomer } from '../repositories/customer.repository'
import type { CreateCustomerInput, Customer } from '../models/customer.model'

export class DomainError extends Error {
  constructor(
    public code: string,
    message: string,
    public statusHint: number,
  ) {
    super(message)
  }
}

export async function registerCustomer(input: CreateCustomerInput): Promise<Customer> {
  const existing = await findCustomerByPhone(input.phone)
  if (existing) {
    throw new DomainError(
      'DUPLICATE_PHONE',
      'A customer with this phone already exists.',
      409,
    )
  }
  return createCustomer(input)
}
```

Rules:
- Every business rule from the spec lives here — not in the route, not in the repository
- Throw `DomainError` with the exact `code` string from `spec-bdd.md`
- `statusHint` carries the intended HTTP status — routes read it, never hardcode status in the service
- Services are pure async functions or small classes — no global state, no side effects beyond DB writes
- Services must be unit-testable without a real DB (repository imports are mocked in unit tests)

---

#### Layer 4 — Route (`src/api/<resource>.routes.ts`)

HTTP adapter only. Validates input, calls the service, formats the response.

```typescript
// src/api/customer.routes.ts
import { Router } from 'express'
import { CreateCustomerSchema } from '../models/customer.model'
import { registerCustomer, DomainError } from '../services/customer.service'
import { requireRole } from '../utils/auth'

const router = Router()

router.post('/', requireRole('RECEPTIONIST', 'MANAGER'), async (req, res) => {
  const parsed = CreateCustomerSchema.safeParse(req.body)

  if (!parsed.success) {
    const fields = parsed.error.errors.map(e => e.path.join('.'))
    return res.status(422).json({
      data: null,
      error: { code: 'VALIDATION_ERROR', message: 'Invalid input.', fields },
    })
  }

  try {
    const customer = await registerCustomer(parsed.data)
    return res.status(201).json({ data: customer, error: null })
  } catch (err) {
    if (err instanceof DomainError) {
      return res.status(err.statusHint).json({
        data: null,
        error: { code: err.code, message: err.message },
      })
    }
    throw err
  }
})

export default router
```

Rules:
- All responses use the `{ data, error }` envelope — no exceptions
- Zod validation happens here, before the service is ever called
- The route never calls Prisma or repository functions directly
- `requireRole(...)` middleware is applied per the allowed roles in the spec
- HTTP status mapping: 201 created, 200 ok, 422 validation, 409 conflict, 401 unauthenticated, 403 wrong role
- Never leak stack traces in error responses — only `code` and `message`

---

#### Layer 5 — Prisma Migration

If the spec introduces or modifies an entity in the data model:

1. Add or update the model in `prisma/schema.prisma` following the entity definitions in `docs/arch/arch-data-model.md`
2. Add DB-level constraints for every uniqueness FR (e.g., `@unique`, `@@unique`)
3. Run: `npx prisma migrate dev --name jira-<id>-<short-description>`

```prisma
// prisma/schema.prisma (append new model)
model Customer {
  id         String    @id @default(uuid())
  name       String
  phone      String    @unique
  email      String?
  createdAt  DateTime  @default(now()) @map("created_at")
  archivedAt DateTime? @map("archived_at")

  @@map("customers")
}
```

Rules:
- Prisma field names: `camelCase` — map to `snake_case` DB columns with `@map`
- Every uniqueness FR → DB-level `@unique` or `@@unique`, not just a service check
- Every FK relation must be explicit in the schema
- Migration name format: `jira-<id>-<short-description>` (e.g., `jira-123-create-customers`)

---

### Step 4 — Write the Tests

Three test files per ticket. Every BDD scenario maps to exactly one `describe` block.

---

#### Unit Tests (`test/unit/<resource>/<action>.test.ts`)

Test the service in isolation. Mock only the repository layer.

```typescript
// test/unit/customers/register.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { registerCustomer, DomainError } from '../../../src/services/customer.service'
import * as repo from '../../../src/repositories/customer.repository'

vi.mock('../../../src/repositories/customer.repository')

describe('JIRA-123 | Scenario 3 — Rejection when phone number already exists', () => {
  beforeEach(() => vi.clearAllMocks())

  it('throws DomainError with DUPLICATE_PHONE when phone is already registered', async () => {
    vi.mocked(repo.findCustomerByPhone).mockResolvedValue({
      id: 'existing-uuid',
      name: 'Existing User',
      phone: '3001234567',
      email: null,
      createdAt: new Date(),
    })

    await expect(
      registerCustomer({ name: 'Carlos Ruiz', phone: '3001234567' }),
    ).rejects.toMatchObject({ code: 'DUPLICATE_PHONE', statusHint: 409 })

    expect(repo.createCustomer).not.toHaveBeenCalled()
  })
})
```

Rules:
- Mock only the repository — never mock the service in unit tests
- `vi.clearAllMocks()` in `beforeEach` of every file
- `describe` label must match `spec-bdd.md` exactly: `'JIRA-NNN | Scenario N — [label]'`
- Test observable behavior (what the service returns/throws), not implementation details

---

#### Integration Tests (`test/integration/<resource>/<action>.test.ts`)

Test service + repository against a real test database. No mocks.

```typescript
// test/integration/customers/register.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { registerCustomer, DomainError } from '../../../src/services/customer.service'
import { prisma } from '../../../src/utils/prisma'

beforeEach(async () => {
  await prisma.customer.deleteMany()
})

describe('JIRA-123 | Scenario 1 — Successful registration with all fields', () => {
  it('creates a customer record and returns a UUID and ISO timestamp', async () => {
    const result = await registerCustomer({
      name: 'Juan Pérez',
      phone: '3001234567',
      email: 'juan@example.com',
    })

    expect(result.id).toMatch(
      /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/,
    )
    expect(result.createdAt).toBeInstanceOf(Date)
    expect(result.name).toBe('Juan Pérez')
  })
})

describe('JIRA-123 | Scenario 3 — Rejection when phone number already exists', () => {
  it('throws DUPLICATE_PHONE and does not create a second record', async () => {
    await registerCustomer({ name: 'First User', phone: '3001234567' })

    await expect(
      registerCustomer({ name: 'Carlos Ruiz', phone: '3001234567' }),
    ).rejects.toMatchObject({ code: 'DUPLICATE_PHONE' })

    const count = await prisma.customer.count()
    expect(count).toBe(1)
  })
})
```

Rules:
- Uses a separate test DB — `DATABASE_URL` from `.env.test`
- `beforeEach` cleans only the tables the test touches — never wipe unrelated tables
- No mocks — the point is to verify real DB interaction
- One `describe` per BDD scenario; keep the same order as `spec-bdd.md`

---

#### E2E Tests (`test/e2e/<resource>/<action>.spec.ts`)

Full HTTP stack via Playwright API testing. One `describe` per BDD scenario, in spec order.

```typescript
// test/e2e/customers/register.spec.ts
import { test, expect } from '@playwright/test'

const BASE = process.env.API_BASE_URL ?? 'http://localhost:3000'

async function getAuthToken(request: any, role = 'RECEPTIONIST'): Promise<string> {
  // log in as a seeded test user for the given role; return JWT
}

test.describe('JIRA-123 | Scenario 1 — Successful registration with all fields', () => {
  test('returns 201 with customer data including UUID and createdAt', async ({ request }) => {
    const token = await getAuthToken(request)
    const res = await request.post(`${BASE}/customers`, {
      headers: { Authorization: `Bearer ${token}` },
      data: { name: 'Juan Pérez', phone: '3001234567', email: 'juan@example.com' },
    })

    expect(res.status()).toBe(201)
    const body = await res.json()
    expect(body.data.name).toBe('Juan Pérez')
    expect(body.data.phone).toBe('3001234567')
    expect(body.data.email).toBe('juan@example.com')
    expect(body.data.id).toMatch(/^[0-9a-f-]{36}$/)
    expect(body.error).toBeNull()
  })
})

test.describe('JIRA-123 | Scenario 7 — Rejection when unauthenticated', () => {
  test('returns 401 with UNAUTHORIZED code when no token is provided', async ({ request }) => {
    const res = await request.post(`${BASE}/customers`, {
      data: { name: 'Test User', phone: '3009999999' },
    })

    expect(res.status()).toBe(401)
    const body = await res.json()
    expect(body.error.code).toBe('UNAUTHORIZED')
    expect(body.data).toBeNull()
  })
})
```

Rules:
- `describe` label = `'[TICKET] | Scenario [N] — [exact scenario label from spec-bdd.md]'`
- Scenarios appear in the file in the same order as `spec-bdd.md`
- Request data values must match the spec verbatim (same names, phones, emails as in the Gherkin)
- Every `Then` assertion in Gherkin → one `expect(...)` call in the test
- For `Scenario Outline` rows → use `test.each` with the Examples table data
- Auth scenarios: send no token — do not send a wrong token unless the spec explicitly tests that case

---

### Step 5 — Register the Route

Mount the new router in the Express app entry point:

```typescript
// src/app.ts
import customerRoutes from './api/customer.routes'
app.use('/customers', customerRoutes)
```

Add the `app.use` line in the correct position. If multiple routes exist, keep them alphabetical by resource name.

---

### Step 6 — Quality Check

Run these verifications before reporting completion. Fix every failure.

**Type check**
```bash
npx tsc --noEmit
```
Zero errors required.

**Unit + integration tests**
```bash
npx vitest run
```
All green. No skipped tests unless the scenario is tagged `@wip` in `spec-bdd.md`.

**E2E tests**
```bash
npx playwright test test/e2e/<resource>/
```
All BDD scenarios pass.

**Manual checklist**
- [ ] Every BDD scenario in `spec-bdd.md` has a `describe` block in the corresponding test file
- [ ] Business rules are in `src/services/` — not in routes or repositories
- [ ] Zod validation is in the route handler — not in the service
- [ ] All responses use `{ data, error }` envelope
- [ ] DB migration exists if the spec touched the data model
- [ ] `@unique` / DB constraints present for every uniqueness FR
- [ ] Auth middleware applied to every route that requires authentication per the spec
- [ ] No fields, endpoints, or behaviors outside the spec were added
- [ ] No `console.log` left in production code
- [ ] No stack traces in error responses

---

### Step 7 — Report Completion

Produce this summary after all checks pass:

```
## JIRA-XXX — Implementation Summary

### Files created / modified
- src/models/<resource>.model.ts
- src/repositories/<resource>.repository.ts
- src/services/<resource>.service.ts
- src/api/<resource>.routes.ts
- src/app.ts  (route registered)
- prisma/migrations/<name>/migration.sql  (if applicable)

### Tests
- test/unit/<resource>/<action>.test.ts        — N unit scenarios
- test/integration/<resource>/<action>.test.ts — N integration scenarios
- test/e2e/<resource>/<action>.spec.ts         — N e2e scenarios (all from spec-bdd.md)

### BDD Coverage
| Scenario | Layers tested | Status |
|---|---|---|
| 1 — [label] | unit + integration + e2e | ✓ |
| ...         | ...                      | ✓ |

### FRs implemented
FR-01 ✓  FR-02 ✓  FR-03 ✓  (list all Must FRs)

### NFRs addressed
| Attribute | How |
|---|---|
| Security | requireRole middleware on POST /customers |
| Performance | No N+1 queries; single DB call per request |
```

---

### Step 8 — Offer Follow-up

> Would you like me to:
> - Create the git branch and open a draft PR (`feat/<TICKET>-<short-description>`)?
> - Generate the React form component for this endpoint?
> - Write an ADR if any non-obvious architectural decision was made?
> - Run a spec-vs-code gap check to verify nothing was missed?

---

## Conventions Quick Reference

| Concern | Convention |
|---|---|
| File names | `kebab-case.ts` |
| Functions / variables | `camelCase` |
| Classes | `PascalCase` |
| Constants | `SCREAMING_SNAKE_CASE` |
| DB tables | `snake_case` plural |
| DB columns | `snake_case` — map with `@map` in Prisma |
| API paths | `kebab-case` plural nouns (`/service-orders`) |
| HTTP 201 | Resource created |
| HTTP 200 | Success with body |
| HTTP 422 | Zod validation failure |
| HTTP 409 | Business rule conflict (duplicate, invalid state) |
| HTTP 401 | Not authenticated (no token) |
| HTTP 403 | Authenticated but wrong role |
| Response envelope | `{ data: T \| null, error: { code, message } \| null }` |
| Error codes | `SCREAMING_SNAKE_CASE` — must match `spec-bdd.md` exactly |
| Test `describe` label | `'JIRA-NNN \| Scenario N — [exact label]'` |
| Prisma migration name | `jira-<id>-<short-description>` |

## Common Pitfalls

| Mistake | Correct approach |
|---|---|
| Validating input in the service | Zod validation belongs only in the route handler |
| Throwing HTTP-specific errors from the service | Throw `DomainError` with `statusHint`; the route converts it to HTTP |
| Calling Prisma from the route directly | Route → Service → Repository — never skip layers |
| Mocking the service in unit tests | Mock only the repository; the service runs real logic |
| Writing tests that depend on prior test state | Each test cleans its own data in `beforeEach` |
| Adding fields or endpoints not in the spec | Out-of-scope items stay out — no gold-plating |
| Skipping the DB migration | Every `@unique` FR needs a DB-level constraint, not just a service check |
| Returning raw Prisma objects from routes | Map to the domain type — never leak `password_hash` or internal Prisma fields |
