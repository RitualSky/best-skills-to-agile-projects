# JIRA-12345 — Spec: Requirements

> **Type:** User Story  
> **Priority:** Must — High  
> **Target version:** Sprint 1  
> **Components:** Auth, API, Users

---

## User Story

**As a** Receptionist, Technician, or Manager  
**I want** to log in with my email and password  
**So that** I can access the system with the permissions corresponding to my role

---

## Context

Currently the TechShop system has no authentication layer. All subsequent features (customer management, service orders, reporting) require knowing who is performing each action — both for role-based access control and for auditability of sensitive operations (e.g., who opened an order, who changed a status).

This story implements the login endpoint and the JWT token that all other protected endpoints will validate via the `requireRole` middleware. It is the foundational security story and must be completed before any other protected story is implemented.

The `users` table already exists in the data model with `email`, `password_hash`, and `role` fields. This story does not create users — it only authenticates existing ones.

---

## Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | The system must accept `email` (required) and `password` (required) as login credentials | Must |
| FR-02 | `email` must be a valid email address format (validated at API boundary) | Must |
| FR-03 | `password` must not be empty (validated at API boundary) | Must |
| FR-04 | The system must verify the password against the stored `password_hash` using bcrypt | Must |
| FR-05 | On success, the system must return a signed JWT token | Must |
| FR-06 | The JWT payload must include: `userId`, `role`, `email`, and `exp` (expiration) | Must |
| FR-07 | If the email is not registered OR the password is wrong, the system must return a single generic error — it must NOT reveal which of the two failed (prevents user enumeration attacks) | Must |
| FR-08 | The success response must include the authenticated user's `id`, `name`, `email`, and `role` so the frontend can initialize the session without a second request | Should |
| FR-09 | The JWT must expire after 8 hours | Should |
| FR-10 | Failed login attempts must be logged with the attempted email, timestamp, and reason (`EMAIL_NOT_FOUND` / `WRONG_PASSWORD`) — in server logs only, never in the API response | Should |

---

## Non-Functional Requirements

| Attribute | Requirement |
|-----------|-------------|
| Performance | Login API response < 300 ms at p95 under 50 concurrent users (bcrypt cost factor must be set accordingly — recommend cost 10) |
| Security | Passwords are never stored or returned in plain text. The generic 401 error prevents email enumeration. JWT must be signed with a secret stored in `JWT_SECRET` env var (min 32 chars). bcrypt cost factor ≥ 10. |
| Auditability | Each failed login attempt must be recorded in server logs with: attempted email, IP address, timestamp, and failure reason. Successful logins must also be logged. |
| Scalability | JWT is stateless — no server-side session storage required. Designed for horizontal scaling. |

---

## Data Model Impact

**Entities affected:** `users`

| Entity | Change | Details |
|--------|--------|---------|
| `users` | No schema change | Table already exists with `id`, `name`, `email` (UNIQUE), `role` (ENUM), `password_hash`. This story only reads from it. |

> No Prisma migration required for this story.  
> ⚠️ **Assumed** — confirm that the `users` table is seeded with at least one test user per role before sprint start.

---

## API Contract

**Endpoint:** `POST /auth/login`  
**Auth:** Public (no token required — this is how the token is obtained)

**Request body:**
```json
{
  "email": "string — required, valid email format",
  "password": "string — required, non-empty"
}
```

**Success response (`200`):**
```json
{
  "data": {
    "token": "string — signed JWT",
    "user": {
      "id": "string — UUID",
      "name": "string",
      "email": "string",
      "role": "RECEPTIONIST | TECHNICIAN | MANAGER"
    }
  },
  "error": null
}
```

**Error responses:**

| Status | `error.code` | When |
|--------|-------------|------|
| 422 | `VALIDATION_ERROR` | `email` is missing/invalid format, or `password` is empty |
| 401 | `INVALID_CREDENTIALS` | Email not found in DB, or password does not match hash |

> Note: Both "email not found" and "wrong password" must return the same `401 / INVALID_CREDENTIALS` response with the same message. Never reveal which condition failed.

---

## Out of Scope

- Password reset / forgot-password flow (future story)
- Token refresh or logout endpoint (future story)
- Multi-factor authentication (MFA)
- OAuth 2.0 / SSO / social login
- Account lockout after N failed attempts (future story)
- User registration / user management (admin feature, separate story)
- Remember-me / long-lived sessions

---

## Acceptance Criteria Summary

> See `spec-bdd.md` for the full Gherkin definitions. The following scenarios must be covered:

1. Successful login with valid email and password — returns 200, JWT token, and user data
2. Rejection when email is not registered — returns 401 with `INVALID_CREDENTIALS`
3. Rejection when password is wrong — returns 401 with `INVALID_CREDENTIALS` (same response as scenario 2)
4. Rejection when email is missing from the request body — returns 422 with `VALIDATION_ERROR`
5. Rejection when email has invalid format — returns 422 with `VALIDATION_ERROR`
6. Rejection when password is empty or missing — returns 422 with `VALIDATION_ERROR`
7. JWT payload verification — returned token contains `userId`, `role`, `email`, and `exp`

---

## Definition of Done

- [ ] `POST /auth/login` endpoint implemented
- [ ] All 7 BDD scenarios pass as automated tests in `test/e2e/`
- [ ] Input validated with Zod schema at the route layer
- [ ] bcrypt verification in the service layer (not in route or repository)
- [ ] JWT signed with `JWT_SECRET` from env — `process.env.JWT_SECRET` must be set; app must fail to start if missing
- [ ] Generic 401 error used for both "email not found" and "wrong password" — confirmed by code review
- [ ] Server-side logging of failed and successful login attempts implemented
- [ ] `JWT_SECRET` documented in `.env.example` (but never committed with a real value)
- [ ] Code reviewed and approved
- [ ] Deployed to staging and smoke-tested
- [ ] Product Owner has accepted the story

---

## Estimation

| Metric | Value |
|--------|-------|
| Story Points | 3 |
| Complexity | Low |
| Uncertainty | Low |
| Suggested size | S |

**Rationale:** Standard auth endpoint with well-understood technology (bcrypt + JWT, both already in the stack). No external integrations. The main complexity driver is the security requirement for generic error responses and bcrypt cost tuning. Similar in scope to JIRA-123 (customer registration), which was also 3 points.

---

## Dependencies

| Type | Item | Status |
|------|------|--------|
| Upstream | `users` table seeded with test users (one per role) | Pending — ⚠️ Assumed: needs confirmation |
| Downstream | JIRA-123 (POST /customers) — requires auth middleware from this story | Pending |
| Downstream | All other protected endpoints — blocked on `requireRole` middleware from this story | Pending |
| External | `JWT_SECRET` env var configured in Railway (staging + prod) | Pending |
