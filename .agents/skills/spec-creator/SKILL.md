# Skill: Spec Creator (spec-creator)

## Description

Generates the complete spec folder for a JIRA ticket (`spec-requirements.md`). Accepts two input modes: a JIRA ticket ID (fetches data via the Jira REST API) or a free-text description typed directly in the prompt. Covers all aspects a spec must have before any BDD scenarios or code can be written.

## Trigger

Use this skill when the user:
- Provides a JIRA ticket ID and says "create spec", "generate spec", "write spec for JIRA-XXX"
- Provides a description and says "create a spec for this", "write the spec", "document this story"
- Needs to produce `spec-requirements.md` before running `bdd-creator` or `apply-spec`

---

## Input Modes

### Mode A — JIRA Ticket ID

The user provides a ticket ID (e.g., `JIRA-124`, `PROJ-42`).

Fetch ticket data from the Jira REST API using the environment variables below.  
If any variable is missing, stop and ask the user to set it before continuing.

| Variable | Purpose |
|---|---|
| `JIRA_BASE_URL` | e.g., `https://yourcompany.atlassian.net` |
| `JIRA_USER_EMAIL` | Jira account email for Basic auth |
| `JIRA_API_TOKEN` | API token from `id.atlassian.com/manage-profile/security/api-tokens` |

**API call:**
```
GET ${JIRA_BASE_URL}/rest/api/3/issue/${TICKET_ID}?expand=renderedFields
Authorization: Basic base64("${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}")
Accept: application/json
```

Use the `WebFetch` tool to make this call. Pass the Authorization header as a base64-encoded string of `email:token`.

### Mode B — Free-Text Description

The user provides a description, rough idea, or partially-formed user story directly in the prompt.  
Skip the API call and use the provided text as the raw input to derive all spec sections.

---

## Instructions

You are a senior product engineer and agile practitioner on the TechShop project.  
Your job is to produce a spec that is complete enough that:
1. A developer can implement it without asking questions about scope.
2. A QA engineer can derive BDD scenarios from it without ambiguity.
3. A reviewer can verify it against the product vision in `docs/features/main.md`.

Always read `docs/features/main.md` to understand product context before generating the spec.

---

### Step 1 — Gather the Input

#### If Mode A (JIRA ticket):

Make the API call and extract these fields from the response:

| Jira field path | Maps to |
|---|---|
| `fields.summary` | Story title |
| `fields.description` (ADF) | Context + raw requirements |
| `fields.issuetype.name` | Story type (User Story / Bug / Task) |
| `fields.priority.name` | Priority context |
| `fields.customfield_10016` or `fields.story_points` | Story points (try both; field ID varies by instance) |
| `fields.labels` | Tags (inform NFRs or context) |
| `fields.components[*].name` | Affected system area |
| `fields.fixVersions[*].name` | Target version or sprint |
| `fields.subtasks[*].key` + `.fields.summary` | Potential task breakdown |
| `fields.issuelinks` | Dependencies (inward = upstream, outward = downstream) |
| `fields.comment.comments[*].body` | Additional context if description is sparse |
| `renderedFields.description` | HTML-rendered description (fallback for parsing) |

**Parsing the Jira description (Atlassian Document Format → Markdown):**

The `fields.description` is an ADF JSON object. Convert it by traversing the node tree:

| ADF node type | Markdown output |
|---|---|
| `paragraph` | Paragraph separated by blank lines |
| `heading` level N | `## ` (N=1), `### ` (N=2), `#### ` (N=3) |
| `bulletList` | `- ` items |
| `orderedList` | `1. ` items |
| `listItem` | Content of the item |
| `text` | Plain text; apply marks: `strong` → `**`, `em` → `_`, `code` → `` ` `` |
| `hardBreak` | `\n` |
| `codeBlock` | ```` ``` ```` fenced block |
| `blockquote` | `> ` prefix |
| `inlineCard` / `mention` | Use the URL or display name as plain text |

If `fields.description` is `null` or empty, check `fields.comment.comments` and `renderedFields.description`.  
If the ticket has almost no content, ask the user to provide more context before continuing.

#### If Mode B (free text):

Read the description literally. Identify:
- Who benefits (actor/role)
- What they want to do (goal)
- Why it matters (business value)
- Any constraints or rules mentioned

Fill gaps with reasonable assumptions and mark them clearly with `> ⚠️ Assumed — confirm with PO`.

---

### Step 2 — Derive the Spec Sections

Before writing, silently derive the following from the gathered input:

| Section | How to derive |
|---|---|
| **User Story** | Reconstruct from summary/description using As a/I want/So that |
| **Context** | 2-4 sentences from description body, business motivation, current pain point |
| **Functional Requirements** | Each distinct capability, rule, or constraint = one FR. Assign MoSCoW priority |
| **Non-Functional Requirements** | Derive from labels, components, security role mentions, performance targets |
| **Data Model Impact** | Does this story create, modify, or delete DB entities? Map to `docs/arch/arch-data-model.md` |
| **API Contract** | HTTP method + path + request shape + response shape (if the story is API-facing) |
| **Out of Scope** | What is explicitly excluded? What adjacent features are NOT being built? |
| **Acceptance Criteria Summary** | Numbered list of scenario groups that spec-bdd.md must cover |
| **Definition of Done** | Start from the project template; add story-specific items |
| **Estimation** | Use story points from Jira or derive from complexity |
| **Dependencies** | From `issuelinks` or explicit mentions in description |

**MoSCoW priority rules for FRs:**

| Priority | When to use |
|---|---|
| **Must** | Without this FR the story cannot be accepted. Core behavior. |
| **Should** | Important but the story can ship without it if time-constrained |
| **Could** | Nice to have; include only if Must/Should are all done |
| **Won't** | Explicitly excluded this sprint (documents scope decisions) |

---

### Step 3 — Write `spec-requirements.md`

Use this exact template. Replace every `[placeholder]`. Remove sections marked optional only if they truly do not apply.

---

```markdown
# [TICKET-ID] — Spec: Requirements

> **Type:** [User Story / Bug Fix / Technical Task]  
> **Priority:** [Must / Should / Could] — [Jira priority if available]  
> **Target version:** [Sprint / Release — from Jira fixVersion, or "TBD"]  
> **Components:** [Affected system area, or "N/A"]

---

## User Story

**As a** [actor — use roles from `docs/features/main.md`: Receptionist, Technician, Manager]  
**I want** [goal — concrete action, not vague intent]  
**So that** [business value — outcome for the actor or the business, not a technical result]

---

## Context

[2–4 sentences. Describe: (1) the current pain point or missing capability, (2) the workflow or journey this story belongs to, (3) why it matters now. Reference related tickets or features by ID if relevant.]

> If any section below contains an assumption, mark it:  
> ⚠️ **Assumed** — confirm with PO before sprint start.

---

## Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | [Concrete, testable capability. One behavior per row. Start with a verb.] | Must |
| FR-02 | [...] | Must |
| FR-03 | [...] | Should |

> Rules for writing FRs:
> - Start with a verb: "The system must…", "The user can…", "The API must return…"
> - Each FR must be independently testable — it maps to at least one BDD scenario
> - Avoid vague words: "fast", "user-friendly", "simple" — use measurable terms

---

## Non-Functional Requirements

| Attribute | Requirement |
|-----------|-------------|
| Performance | [e.g., API response < 300 ms at p95 under 50 concurrent users. Remove if not applicable.] |
| Security | [e.g., Endpoint requires authenticated session. Allowed roles: RECEPTIONIST, MANAGER.] |
| Accessibility | [e.g., Form fields must have visible labels and ARIA attributes (WCAG 2.1 AA). Remove if not applicable.] |
| Auditability | [e.g., Each write operation must record acting user ID and timestamp. Remove if not applicable.] |
| Scalability | [Remove if not applicable.] |

> Remove rows that do not apply. Do not leave placeholder text — either fill it in or remove the row.

---

## Data Model Impact

[Does this story touch the database schema?]

**Entities affected:** [List DB tables from `docs/arch/arch-data-model.md`, or "None"]

| Entity | Change | Details |
|--------|--------|---------|
| `[table_name]` | [New / Modify / No change] | [New fields, constraints, or indices. Use snake_case column names.] |

> If no DB changes are needed, write: "No data model changes required for this story."

---

## API Contract

[Fill this section only for stories that expose or consume an HTTP endpoint.]

**Endpoint:** `[HTTP METHOD] /[resource-path]`  
**Auth:** [Required roles, or "Public"]

**Request body:**
```json
{
  "[field]": "[type — required/optional, constraints]"
}
```

**Success response (`[HTTP status]`):**
```json
{
  "data": { "[field]": "[type]" },
  "error": null
}
```

**Error responses:**
| Status | `error.code` | When |
|--------|-------------|------|
| 422 | `VALIDATION_ERROR` | Input fails schema validation |
| 409 | `[BUSINESS_RULE_CODE]` | [When the business rule is violated] |
| 401 | `UNAUTHORIZED` | No valid session token |
| 403 | `FORBIDDEN` | Valid token but insufficient role |

> If this story has no API surface (pure UI, pure infra, etc.), write: "No API contract for this story."

---

## Out of Scope

> Be explicit. Listing exclusions prevents false BDD scenarios and scope creep.

- [What is NOT being built in this story. Reference a future ticket ID if known.]
- [Another exclusion]

---

## Acceptance Criteria Summary

> This section is the input to `bdd-creator`. Each numbered item below becomes one or more Gherkin scenarios.  
> See `spec-bdd.md` for the full Gherkin definitions.

1. [Happy path — main success flow with all fields valid]
2. [Alternative path — optional fields omitted or alternate valid input]
3. [Business rule violation — e.g., duplicate, invalid state transition]
4. [Validation error — missing required field]
5. [Validation error — invalid format]
6. [Security — unauthenticated access rejected]
7. [Security — wrong role rejected, if applicable]
[Add or remove items based on the FRs above]

---

## Definition of Done

- [ ] All `spec-bdd.md` scenarios pass as automated tests in `test/e2e/`
- [ ] Unit tests written and passing (service layer, coverage ≥ 80%)
- [ ] Integration tests verify DB constraints and repository behavior
- [ ] Input validated with Zod schema at the route layer
- [ ] Business rules enforced in the service layer
- [ ] [If DB change:] Prisma migration created and applied
- [ ] [If DB change:] DB-level constraints present for all uniqueness/FK rules
- [ ] Endpoint documented in API reference (if applicable)
- [ ] Code reviewed and approved
- [ ] Deployed to staging and smoke-tested
- [ ] Product Owner has accepted the story

---

## Estimation

| Metric | Value |
|--------|-------|
| Story Points | [1 / 2 / 3 / 5 / 8 / 13] |
| Complexity | [Low / Medium / High] |
| Uncertainty | [Low / Medium / High] |
| Suggested size | [XS / S / M / L / XL] |

**Rationale:** [1-2 sentences. Name the main complexity drivers: number of layers touched, external integrations, new DB schema, tricky business rules, accessibility requirements, etc. Reference a similar past story if helpful.]

---

## Dependencies

| Type | Item | Status |
|------|------|--------|
| Upstream | [Story/epic that must be done before this one, with JIRA ID] | [Done / In Progress / Pending] |
| Downstream | [Story that is blocked by this one, with JIRA ID] | [Pending] |
| External | [Third-party API, design file, access needed, infra change] | [Pending] |

> Leave empty if there are no known dependencies: "No dependencies identified."
```

---

### Step 4 — Save the File

1. Create the directory `specs/<TICKET-ID>/` if it does not exist.
2. Write the output to `specs/<TICKET-ID>/spec-requirements.md`.
3. Do NOT create `spec-bdd.md` — that is the job of the `bdd-creator` skill.

---

### Step 5 — Quality Check

Verify every item before delivering. Fix issues before showing the file.

**Completeness**
- [ ] User Story has all three parts (As a / I want / So that) and the "So that" describes business value, not a technical outcome
- [ ] Context is 2–4 sentences — no single-line context, no wall of text
- [ ] Every FR is independently testable and starts with a verb
- [ ] At least one FR addresses each of: input validation, business rule, success output
- [ ] NFRs with "Remove if not applicable" were either filled or removed — no placeholder text remains
- [ ] Data Model Impact is not empty — either lists affected entities or explicitly says "no changes"
- [ ] API Contract is present for any story that creates or modifies an endpoint
- [ ] Out of Scope has at least one item (if nothing is out of scope, that itself is suspicious — confirm with the user)
- [ ] Acceptance Criteria Summary has at least one happy-path item and at least one negative/error item
- [ ] DoD includes story-specific items beyond the generic checklist
- [ ] Estimation has a written rationale (not just numbers)

**Correctness**
- [ ] Roles used in NFRs / API Contract match the project roles: `RECEPTIONIST`, `TECHNICIAN`, `MANAGER`
- [ ] DB table/column names follow `snake_case` conventions from `docs/arch/arch-data-model.md`
- [ ] HTTP status codes match project API conventions (422, 409, 401, 403, 201, 200)
- [ ] Error codes are `SCREAMING_SNAKE_CASE`
- [ ] No feature described in "Out of Scope" appears in the FRs

**Traceability**
- [ ] Ticket ID in the file name and in the `#` heading match the requested ticket
- [ ] Every FR has a corresponding item in Acceptance Criteria Summary (1:1 or many FRs → one scenario group)
- [ ] Upstream dependencies reference real JIRA ticket IDs (from `issuelinks` or the user's input)

**Jira-specific (Mode A only)**
- [ ] All fields used were sourced from the API response — no invented data
- [ ] Any section with an assumption is marked with `⚠️ Assumed`
- [ ] Story points match the Jira estimate if one exists

---

### Step 6 — Report and Offer Follow-up

After saving, report:

```
## Spec created: specs/[TICKET-ID]/spec-requirements.md

### Summary
- Story type: [User Story / Bug / Task]
- Actor: [role]
- FRs generated: [N] (Must: N, Should: N, Could: N)
- NFRs: [list attributes covered]
- Data model: [No changes / N entities affected]
- API surface: [None / METHOD /path]
- Estimation: [N] points ([Complexity] complexity, [Uncertainty] uncertainty)
- Assumptions made: [N — list them] / None

### Next steps
```

Then ask:

> Would you like me to:
> - Run **`bdd-creator`** to generate `spec-bdd.md` from this requirements file?
> - Run **`enrich-us`** to review the user story format and acceptance criteria for completeness?
> - Open a Jira sub-task for each item in the Acceptance Criteria Summary?
> - Translate this spec to another language?

---

## Jira API — Field Reference

Some Jira custom field IDs vary by instance. Try these in order; use the first one that returns a value:

| Data | Common field paths |
|---|---|
| Story points | `fields.customfield_10016`, `fields.customfield_10028`, `fields.story_points` |
| Acceptance criteria | `fields.customfield_10500`, `fields.customfield_10014`, look for a field named "Acceptance Criteria" in the response keys |
| Epic link | `fields.customfield_10014`, `fields.customfield_10008` |
| Sprint | `fields.customfield_10020[0].name` |

If a field returns `null`, skip it silently — do not add an empty row to the spec.

---

## ADF Parsing — Extended Node Reference

For complex Jira descriptions, handle these additional ADF nodes:

| ADF node | Markdown |
|---|---|
| `table` → `tableRow` → `tableCell` / `tableHeader` | GFM table: `\| col \| col \|` with separator row |
| `media` (image attachment) | `> 📎 [Attachment: filename]` |
| `expand` (collapsible section) | Include the title as `#### ` heading + content |
| `status` (Jira status lozenge) | Inline text: `[STATUS_NAME]` |
| `emoji` | Unicode character if known; otherwise remove |
| `rule` (horizontal rule) | `---` |
| Nested `bulletList` inside `listItem` | Indent with 2 spaces |

---

## Example Invocations

### Mode A — JIRA ticket
**Input:** `create spec for JIRA-124`  
**What happens:**
1. Fetches `${JIRA_BASE_URL}/rest/api/3/issue/JIRA-124`
2. Parses ADF description, extracts summary, components, priority, story points, linked issues
3. Reads `docs/features/main.md` for product context
4. Generates `specs/JIRA-124/spec-requirements.md` with all 10 sections filled
5. Reports: 5 FRs (Must), 2 NFRs, no DB changes, 1 upstream dependency (JIRA-123), estimate 3 pts

### Mode B — Free text
**Input:** `create a spec for: we need the technician to be able to update the status of a service order`  
**What happens:**
1. Derives actor = Technician, goal = update order status, entity = service_orders
2. Reads `docs/arch/arch-data-model.md` to find the state machine and existing DB schema
3. Derives FRs from the state machine transitions: PENDING → IN_PROGRESS → DONE → DELIVERED + CANCELLED
4. Generates `specs/JIRA-TBD/spec-requirements.md` with `⚠️ Assumed` markers on missing details
5. Asks user to confirm the ticket ID before saving
