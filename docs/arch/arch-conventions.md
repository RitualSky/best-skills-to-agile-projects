# Architecture: Conventions

## Naming

| Artifact | Convention | Example |
|----------|-----------|---------|
| Files | kebab-case | `customer-service.ts` |
| Classes | PascalCase | `CustomerService` |
| Functions / variables | camelCase | `findByPhone` |
| Constants | SCREAMING_SNAKE | `MAX_SEARCH_RESULTS` |
| Database tables | snake_case plural | `service_orders` |
| Database columns | snake_case | `estimated_price` |
| API routes | kebab-case plural nouns | `GET /service-orders/:id` |
| Env variables | SCREAMING_SNAKE | `DATABASE_URL` |

## API Conventions

- REST. Nouns in URLs, verbs via HTTP method.
- All responses wrapped in `{ data, error, meta }`.
- HTTP 422 for validation errors with field-level messages.
- HTTP 409 for business rule conflicts (e.g., duplicate phone).
- Timestamps always in ISO 8601 UTC.

```json
// Success
{ "data": { "id": "abc", "name": "Juan Pérez" }, "error": null }

// Error
{ "data": null, "error": { "code": "DUPLICATE_PHONE", "message": "A customer with this phone already exists." } }
```

## Git Conventions

- Branch naming: `feat/JIRA-123-register-customer`, `fix/JIRA-130-search-crash`
- Commit format: `feat(JIRA-123): add customer registration endpoint`
- One PR per user story. PRs must link to the JIRA ticket.
- No merging without green CI and one approval.

## Spec Mapping Rule

Every BDD scenario in `docs/specs/<TICKET>/spec-bdd.md` must have a corresponding test in `test/e2e/`. The test file must reference the scenario number in its `describe` block:

```ts
describe('JIRA-123 | Scenario 1 — Register a new customer', () => { ... })
```

This creates a traceable chain: **spec → test → code**.

## Code Review Checklist

- [ ] Does the implementation match the spec exactly? No gold-plating.
- [ ] Is every BDD scenario covered by a test?
- [ ] Are business rules enforced in the service layer, not the route handler?
- [ ] Are all new fields validated with Zod at the API boundary?
- [ ] No secrets or credentials in the codebase.
