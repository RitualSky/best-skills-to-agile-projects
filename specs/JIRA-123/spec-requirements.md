# JIRA-123 — Spec: Requirements

## User Story

**As a** receptionist  
**I want** to register a new customer with their name, phone number, and email  
**So that** I can link future service orders to them and contact them when their device is ready

## Context

Currently the shop records customer data in a paper notebook. When a customer returns, the receptionist must search through physical records, which causes delays during busy hours. This story introduces the digital customer registry as the foundation for all other features.

## Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | The system must accept name (required), phone (required), and email (optional) | Must |
| FR-02 | Phone number must be unique across all customers | Must |
| FR-03 | If a phone number already exists, the system must reject the registration and return a clear error | Must |
| FR-04 | Name must be 2–100 characters | Must |
| FR-05 | Phone must match the format: digits only, 7–15 characters | Must |
| FR-06 | Email, when provided, must be a valid email address | Must |
| FR-07 | On success, the system must return the new customer ID and timestamp | Must |
| FR-08 | The registration form must be accessible via keyboard (no mouse required) | Should |

## Non-Functional Requirements

| Attribute | Requirement |
|-----------|-------------|
| Performance | Registration API response < 300 ms at p95 under 50 concurrent users |
| Security | Endpoint requires authenticated session (RECEPTIONIST or MANAGER role) |
| Accessibility | Form fields must have visible labels and ARIA attributes (WCAG 2.1 AA) |
| Auditability | Each creation must record the acting user ID and timestamp in the DB |

## Out of Scope for this Story

- Editing customer information (JIRA-128)
- Duplicate detection by name similarity (future enhancement)
- Bulk import of customers from CSV

## Acceptance Criteria Summary

See `spec-bdd.md` for the full BDD scenario definitions. The following scenarios must be covered:

1. Successful registration with all fields
2. Successful registration without email (email optional)
3. Rejection when phone number already exists
4. Rejection when name is missing
5. Rejection when phone format is invalid
6. Rejection when email format is invalid
7. Rejection when unauthenticated

## Definition of Done

- [ ] API endpoint `POST /customers` implemented
- [ ] All 7 BDD scenarios pass as automated tests in `test/e2e/`
- [ ] Input validated with Zod schema at the route layer
- [ ] Business rule (unique phone) enforced in the service layer
- [ ] Database constraint (UNIQUE on phone) present in Prisma migration
- [ ] Endpoint documented in API reference
- [ ] Code reviewed and merged to main
- [ ] Product Owner has verified on staging

## Estimation

| Metric | Value |
|--------|-------|
| Story Points | 3 |
| Complexity | Low |
| Uncertainty | Low |

**Rationale:** Standard CRUD endpoint with one business rule. No external integrations. Similar to past registration stories. 3 points accounts for the accessibility requirement and e2e test setup.
