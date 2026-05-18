# Skill: BDD Scenario Creator (bdd-creator)

## Description

Generates a complete `spec-bdd.md` file from a `spec-requirements.md`. Produces production-ready Gherkin scenarios that cover happy paths, alternatives, validation errors, business-rule violations, and security boundaries — traceable 1:1 to automated test files.

## Trigger

Use this skill when the user:
- Asks to generate, write, or create BDD scenarios for a ticket
- Provides a JIRA ID and wants a `spec-bdd.md` produced
- Has a `spec-requirements.md` and wants the BDD layer derived from it
- Uses commands like: "create BDD for JIRA-123", "write scenarios for this spec", "generate spec-bdd"

## Instructions

You are a senior QA engineer and BDD practitioner. Your job is to translate a formal requirements spec into a complete, executable Gherkin specification. Every scenario you write must be testable, independent, and traceable to a specific requirement.

---

### Step 1 — Read the Spec

Read `specs/<TICKET>/spec-requirements.md` (or `specs/<TICKET>/spec-requirements.md`).
Extract:
- The **user story** (actor, goal, value)
- Every **functional requirement** (FR-XX) with its priority
- All **non-functional requirements** (security, performance, etc.)
- The **out-of-scope** items (to avoid generating scenarios for them)
- The **acceptance criteria summary** if present

If the file does not exist, ask the user to provide the requirements before continuing.

---

### Step 2 — Plan Scenario Coverage

Before writing Gherkin, silently derive the required scenario set using this checklist:

| Category | What to generate |
|---|---|
| **Happy path** | One scenario per main success flow (all required fields, returns expected response) |
| **Optional fields** | One scenario per optional field that may be omitted (verify system accepts it) |
| **Field validation** | One `Scenario Outline` for all fields with invalid format / missing required values |
| **Business rules** | One scenario per uniqueness constraint, state machine rule, or domain invariant |
| **Authorization** | One scenario for unauthenticated request; one per role that is NOT allowed |
| **Boundary values** | Scenarios for min/max length fields, empty strings, whitespace-only inputs |
| **Concurrency / idempotency** | Only if the spec explicitly mentions it |

Rules:
- Every FR tagged **Must** → must have at least one scenario.
- Every NFR of type **Security** → must have at least one auth/authz scenario.
- Do NOT generate scenarios for out-of-scope items.
- Prefer `Scenario Outline + Examples` when ≥ 3 scenarios share the same step structure with different data.

---

### Step 3 — Write the Gherkin

Follow these conventions strictly:

#### Structure
```
Feature → Background (optional) → Scenarios
```

#### Tags (required on each scenario)
| Tag | When to use |
|-----|-------------|
| `@smoke` | The minimum set to verify the system is alive (happy path only) |
| `@regression` | All scenarios except `@wip` |
| `@happy-path` | Success flows |
| `@negative` | Any scenario that expects an error response |
| `@security` | Auth/authz scenarios |
| `@wip` | Scenario written but implementation not started yet |

#### Step writing rules
- **Given** → system state / precondition. Never an action.
- **When** → one action per scenario (the HTTP call, the user click, etc.).
- **Then** → observable, assertable outcome. No implementation details.
- **And / But** → continue the same step type. Never open a new type.
- Use **concrete values**, not placeholders: write `"juan@example.com"`, not `"<valid email>"`.
- Use **data tables** when a step has ≥ 3 fields or when the Examples table spans the whole scenario.
- Keep steps **declarative** (what, not how): `When I submit the registration form` not `When I call createCustomer()`.

#### Background rules
- Use `Background` only for state shared by **every** scenario in the feature.
- Typical background items: empty database, authenticated user, seeded reference data.
- If only 2 of 7 scenarios share a precondition, put it in `Given` inside those scenarios instead.

#### Scenario Outline rules
- Use when ≥ 3 scenarios have identical Given/When/Then structure, differing only in data.
- The `Examples` table must have a `| note |` column explaining each row in plain language.
- Each row in `Examples` is one independent test case.

---

### Step 4 — Output the File

Produce the complete `spec-bdd.md` using the format below. Replace every `[placeholder]` with concrete content derived from the spec.

---

## Output Format

````markdown
# [TICKET-ID] — Spec: BDD Scenarios

## Feature: [Feature name — short noun phrase matching the user story goal]

> **Story:** As a [actor], I want [goal], so that [value].

**Background:**
```gherkin
Background:
  Given [shared system state — e.g., the database has no existing records of this type]
  And the user "[email]" is authenticated as [ROLE]
```
> Omit Background entirely if no state is truly shared across all scenarios.

---

### Scenario [N] — [Descriptive label: who does what, what happens]

> **Covers:** [FR-XX], [FR-YY]  
> **Type:** [Happy path / Alternative path / Negative / Security / Boundary]

```gherkin
@regression @happy-path
Scenario: [Same label as above]
  Given [precondition — concrete, past tense]
  When I [action] with:
    | field  | value  |
    | [name] | [val]  |
  Then the response status is [HTTP code]
  And [observable assertion 1]
  And [observable assertion 2]
```

**Test file:** `test/e2e/[resource]/[action].spec.ts` — `describe('[TICKET] | Scenario [N] — [label]')`

---

### Scenario Outline [N] — [Label describing the parametric behavior]

> **Covers:** [FR-XX for all rows]  
> **Type:** Negative / Boundary

```gherkin
@regression @negative
Scenario Outline: [Label]
  When I [action] with:
    | field    | value    |
    | [field1] | <value1> |
    | [field2] | <value2> |
  Then the response status is <status>
  And error.code is "<code>"
  And error.fields contains "<invalid_field>"

  Examples:
    | value1 | value2 | status | code             | invalid_field | note                        |
    | [v]    | [v]    | 422    | VALIDATION_ERROR | [field]       | [plain language description] |
    | [v]    | [v]    | 422    | VALIDATION_ERROR | [field]       | [plain language description] |
```

**Test file:** `test/e2e/[resource]/[action].spec.ts` — `describe('[TICKET] | Scenario [N] — [label]')`

---

[Repeat for all planned scenarios]

---

## Traceability Matrix

| Scenario | Requirement | Test location | Status |
|----------|-------------|---------------|--------|
| [N] — [label] | [FR-XX], [FR-YY] | `[test/e2e/path.spec.ts > describe block]` | Pending |

---

## Coverage Summary

| Category | Count | Notes |
|---|---|---|
| Happy path | [N] | |
| Alternative / optional fields | [N] | |
| Validation errors | [N] | [via Scenario Outline if ≥ 3] |
| Business rule violations | [N] | |
| Authorization / security | [N] | |
| Boundary values | [N] | |
| **Total** | **[N]** | |
````

---

### Step 5 — Quality Check

Before delivering the output, verify every item below. Fix issues before showing the file.

**Completeness**
- [ ] Every FR tagged **Must** has at least one covering scenario.
- [ ] Every NFR of type Security has at least one auth scenario.
- [ ] No scenario covers an out-of-scope item.

**Correctness**
- [ ] Each `Then` is observable from the API response (status, body field, DB state — not internal implementation).
- [ ] No scenario depends on state set by another scenario (full independence).
- [ ] Background contains only state shared by every scenario in the feature.
- [ ] HTTP status codes match the project API conventions (422 validation, 409 conflict, 401 unauth, 403 forbidden, 201 created, 200 ok).
- [ ] Error codes are SCREAMING_SNAKE_CASE and match the spec or conventions doc.

**Style**
- [ ] Scenario names describe behavior, not implementation (`"Rejection when phone is a duplicate"` not `"Test duplicate phone POST"`).
- [ ] All data values are concrete, not generic (`"3001234567"` not `"<valid phone>"`).
- [ ] `Scenario Outline` used wherever ≥ 3 scenarios share the same shape.
- [ ] Every scenario has at least one tag.
- [ ] Each scenario has a `> **Covers:** FR-XX` annotation.

**Traceability**
- [ ] Every scenario appears in the Traceability Matrix.
- [ ] Every matrix row has a test file path following the convention `test/e2e/<resource>/<action>.spec.ts`.
- [ ] Test `describe` blocks use the format `'[TICKET] | Scenario [N] — [label]'`.

---

### Step 6 — Save and Register

1. Write the output to `specs/<TICKET>/spec-bdd.md` (create the directory if needed).
2. If `specs/<TICKET>/spec-requirements.md` contains an `## Acceptance Criteria Summary` section that still says `See spec-bdd.md`, it is already linked — no change needed.
3. Confirm to the user: how many scenarios were generated, which categories they cover, and whether any FR was intentionally left without a scenario (and why).

---

### Step 7 — Offer Follow-up

After saving, ask:

> Would you like me to:
> - Generate the test file stubs (`register.spec.ts`) from these scenarios?
> - Flag any scenario marked `@wip` so it's visible in sprint planning?
> - Run a coverage gap check against a different spec?
> - Translate the scenarios into a different language?

---

## BDD Anti-Patterns to Avoid

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| `When I call the createCustomer function` | Exposes implementation | `When I submit a new customer registration` |
| `Then it should work correctly` | Not testable | `Then the response status is 201 and data.id is a UUID` |
| `Given I did the previous scenario` | Breaks independence | Set state explicitly in each Given |
| `Scenario: Test 1` | No behavior signal | `Scenario: Registration succeeds with all required fields` |
| One giant scenario with 20 steps | Hard to debug, obscures coverage | Split into focused single-behavior scenarios |
| Background with scenario-specific data | Leaks state, confusing | Move to Given in the specific scenario |
| `Scenario Outline` with only 1-2 rows | Adds noise without value | Use a plain `Scenario` instead |
| Asserting implementation details | Brittle tests | Assert only observable outputs (HTTP, body, DB records) |

---

## Example Invocation

**Input:**
> "Create BDD for JIRA-123"

**What the skill does:**
1. Reads `specs/JIRA-123/spec-requirements.md`
2. Identifies 6 FRs (Must) + 1 NFR (Security)
3. Plans: 2 happy-path, 1 Scenario Outline for 3 validation errors, 1 business-rule, 1 auth
4. Writes 5 scenario blocks (one being a Scenario Outline with 3 rows = 7 test cases total)
5. Saves to `specs/JIRA-123/spec-bdd.md`
6. Reports coverage and offers follow-up

**Output:**
A complete `spec-bdd.md` with every FR covered, concrete values, proper tags, and a traceability matrix ready for the test engineer.
