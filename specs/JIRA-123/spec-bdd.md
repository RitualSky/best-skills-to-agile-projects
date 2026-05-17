# JIRA-123 — Spec: BDD Scenarios

## Feature: Customer Registration

**Background:**
- Given the system has no customers registered
- And the user "maria@techshop.com" is authenticated as RECEPTIONIST

---

### Scenario 1 — Successful registration with all fields

```gherkin
Given I have a valid customer payload:
  | field | value               |
  | name  | Juan Pérez          |
  | phone | 3001234567          |
  | email | juan@example.com    |
When I POST /customers with that payload
Then the response status is 201
And the response body contains:
  | field      | value         |
  | data.name  | Juan Pérez    |
  | data.phone | 3001234567    |
  | data.email | juan@example.com |
And data.id is a valid UUID
And data.createdAt is a valid ISO 8601 timestamp
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 1')`

---

### Scenario 2 — Successful registration without email

```gherkin
Given I have a customer payload with no email:
  | field | value      |
  | name  | Ana Torres  |
  | phone | 3109876543 |
When I POST /customers with that payload
Then the response status is 201
And data.email is null
And data.id is a valid UUID
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 2')`

---

### Scenario 3 — Rejection when phone number already exists

```gherkin
Given a customer with phone "3001234567" already exists in the system
When I POST /customers with:
  | field | value       |
  | name  | Carlos Ruiz  |
  | phone | 3001234567  |
Then the response status is 409
And error.code is "DUPLICATE_PHONE"
And error.message contains "phone"
And no new customer record is created in the database
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 3')`

---

### Scenario 4 — Rejection when name is missing

```gherkin
When I POST /customers with:
  | field | value      |
  | phone | 3001111111 |
Then the response status is 422
And error.code is "VALIDATION_ERROR"
And error.fields contains "name"
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 4')`

---

### Scenario 5 — Rejection when phone format is invalid

```gherkin
When I POST /customers with:
  | field | value        |
  | name  | Luis Méndez  |
  | phone | abc-wrong    |
Then the response status is 422
And error.fields contains "phone"
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 5')`

---

### Scenario 6 — Rejection when email format is invalid

```gherkin
When I POST /customers with:
  | field | value         |
  | name  | Rosa Gómez    |
  | phone | 3002222222    |
  | email | not-an-email  |
Then the response status is 422
And error.fields contains "email"
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 6')`

---

### Scenario 7 — Rejection when unauthenticated

```gherkin
Given I am NOT authenticated (no token in request)
When I POST /customers with a valid payload
Then the response status is 401
And error.code is "UNAUTHORIZED"
```

**Test file:** `test/e2e/customers/register.spec.ts` — `describe('Scenario 7')`

---

## Traceability Matrix

| Scenario | Requirement | Test | Status |
|----------|-------------|------|--------|
| 1 | FR-01, FR-07 | `register.spec.ts > Scenario 1` | Pending |
| 2 | FR-01 (email optional) | `register.spec.ts > Scenario 2` | Pending |
| 3 | FR-02, FR-03 | `register.spec.ts > Scenario 3` | Pending |
| 4 | FR-04 | `register.spec.ts > Scenario 4` | Pending |
| 5 | FR-05 | `register.spec.ts > Scenario 5` | Pending |
| 6 | FR-06 | `register.spec.ts > Scenario 6` | Pending |
| 7 | NFR Security | `register.spec.ts > Scenario 7` | Pending |
