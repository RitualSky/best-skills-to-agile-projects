# Skill: Docs Creator (docs-creator)

## Description

Reads the entire codebase (source, tests, config files, ORM schema) and generates or regenerates the `docs/` folder with two subfolders: `arch/` (architecture documentation) and `features/` (product feature documentation). Produces the same document structure used in this workspace. Idempotent — safe to run on an existing `docs/` folder.

## Trigger

Use this skill when the user:
- Asks to generate, create, or bootstrap the docs for a project
- Says "generate docs", "create docs folder", "document this codebase", "bootstrap docs"
- Has a codebase with no `docs/` folder and wants to create one from scratch
- Has an existing `docs/` that is outdated and wants to regenerate it

---

## Output Structure

```
docs/
  arch/
    arch-stack-tech.md     ← technology stack, folder structure, ADR log
    arch-conventions.md    ← naming rules, API conventions, git workflow, spec mapping
    arch-diagrams.md       ← C4 diagrams, entity overview, sequence, state, reference to data model
    arch-data-model.md     ← entities, fields, constraints, enums, state machines, spec writing guide
  features/
    main.md                ← product vision, user roles, feature map, MVP scope
    feature-<domain>.md    ← one file per domain area (one per resource group)
```

---

## Instructions

You are a senior technical writer and software architect. Your job is to reverse-engineer accurate, useful documentation directly from the codebase. Do not invent — derive from the code. If something cannot be determined from the code, write a placeholder marked `> ⚠️ Not determinable from code — fill in manually`.

---

### Step 1 — Discover the Codebase Structure

Run these Glob patterns to understand what exists. Adjust paths if the project uses non-standard folder names.

| What to find | Glob pattern |
|---|---|
| Package manifest | `package.json`, `pyproject.toml`, `pom.xml`, `go.mod`, `Cargo.toml` |
| Type / compiler config | `tsconfig.json`, `tsconfig.*.json` |
| ORM / DB schema | `prisma/schema.prisma`, `*.sql`, `migrations/**/*.sql`, `models.py`, `entities/**/*.ts` |
| CI config | `.github/workflows/*.yml`, `Dockerfile`, `docker-compose.yml`, `.gitlab-ci.yml` |
| Source root | `src/**/*.ts`, `src/**/*.py`, `app/**/*.rb`, `internal/**/*.go` |
| Route / controller files | `src/api/*.ts`, `src/routes/*.ts`, `src/controllers/**/*.ts`, `routes/**/*.py` |
| Service / use-case files | `src/services/**/*.ts`, `src/use-cases/**/*.ts`, `services/**/*.py` |
| Repository / data-access files | `src/repositories/**/*.ts`, `repositories/**/*.py`, `src/dao/**/*.ts` |
| Model / schema / type files | `src/models/**/*.ts`, `src/schemas/**/*.ts`, `src/types/**/*.ts` |
| Test files | `test/**/*.spec.ts`, `test/**/*.test.ts`, `**/__tests__/**/*.ts`, `tests/**/*.py` |
| Environment template | `.env.example`, `.env.template` |
| Existing README | `README.md`, `readme.md` |

After globbing, read the following files in full (they are the primary sources for documentation):
1. `package.json` (or equivalent manifest)
2. `prisma/schema.prisma` (or equivalent ORM schema)
3. All route/controller files
4. All service files
5. All model/schema files
6. All test files (scan for `describe` block labels — they reveal feature names and scenario coverage)
7. `.env.example`

---

### Step 2 — Determine docs/ State

| Situation | Action |
|---|---|
| `docs/` does not exist | Create `docs/arch/` and `docs/features/` and generate all files |
| `docs/` exists, `arch/` and `features/` are missing | Create both subfolders and generate all files |
| `docs/` exists with `arch/` and/or `features/` | Regenerate each file; preserve manually-added content by merging carefully |

For existing files: read the current content first. Keep sections that are clearly manual (e.g., ADR decisions, known constraints written in prose) and update only the sections derivable from code.

---

### Step 3 — Generate `docs/arch/`

Generate four files. Use the exact file names below.

---

#### `docs/arch/arch-stack-tech.md`

**Derived from:** `package.json` dependencies, folder structure, CI config.

```markdown
# Architecture: Technology Stack

## Guiding Principles

[Derive from the code's structure: e.g., if there is a strict layered folder structure → "Layered architecture enforced by folder convention". If there are no principles visible from code, mark as ⚠️]

## Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| [e.g., Frontend] | [e.g., React 18 + TypeScript] | [Derive from package.json] |
| [e.g., Backend] | [e.g., Node.js + Express] | [Derive from package.json] |
| [e.g., Database] | [e.g., PostgreSQL] | [Derive from ORM schema / docker-compose] |
| [e.g., ORM] | [e.g., Prisma] | [Derive from package.json + schema file] |
| [e.g., Auth] | [e.g., JWT + bcrypt] | [Derive from package.json] |
| [e.g., Testing (unit)] | [e.g., Vitest] | [Derive from devDependencies] |
| [e.g., Testing (e2e)] | [e.g., Playwright] | [Derive from devDependencies] |
| [e.g., CI] | [e.g., GitHub Actions] | [Derive from .github/workflows/] |

**How to derive the stack table:**
- `package.json` `dependencies` → identify the main runtime framework, ORM, auth libraries
- `devDependencies` → identify test runners, linters, build tools
- `docker-compose.yml` / `.env.example` → identify database type and version
- `.github/workflows/*.yml` → identify CI platform and Node/Python/Go version

## Folder Structure

[Derive from the actual directory tree. Use the Glob results from Step 1 to build an annotated tree.]

\`\`\`
src/
  [folder]/   [what this folder is for — derive from the files inside it]
  ...
test/
  [folder]/   [derive from test file names]
\`\`\`

## Decision Log

| Decision | Chosen | Rejected | Reason |
|----------|--------|----------|--------|
[Derive only if ADR files exist (docs/adr/, decisions/) or if comments in code explain technology choices. Otherwise: ⚠️ Fill in manually.]
```

---

#### `docs/arch/arch-conventions.md`

**Derived from:** actual file names, function names, API route patterns in route files, response shapes in route handlers, git log for commit format, test describe block labels.

```markdown
# Architecture: Conventions

## Naming

[Derive by scanning actual file names and function/class names in src/]

| Artifact | Convention | Example |
|----------|-----------|---------|
| Files | [e.g., kebab-case] | [real filename from codebase] |
| Classes | [e.g., PascalCase] | [real class name from codebase] |
| Functions / variables | [e.g., camelCase] | [real function name from codebase] |
| Constants | [e.g., SCREAMING_SNAKE] | [real constant from codebase] |
| Database tables | [e.g., snake_case plural] | [from ORM schema] |
| Database columns | [e.g., snake_case] | [from ORM schema] |
| API routes | [e.g., kebab-case plural nouns] | [from route files] |
| Env variables | [e.g., SCREAMING_SNAKE] | [from .env.example] |

## API Conventions

[Derive from actual route handlers: response envelope shape, HTTP status codes used, error code format]

- [e.g., REST. Nouns in URLs, verbs via HTTP method.]
- [e.g., All responses wrapped in `{ data, error, meta }`.]
- [e.g., HTTP 422 for validation errors.]
- [e.g., HTTP 409 for business rule conflicts.]
- [e.g., Timestamps in ISO 8601 UTC.]

\`\`\`json
// Success — derive from actual route response
{ "data": { ... }, "error": null }

// Error — derive from actual error handling
{ "data": null, "error": { "code": "...", "message": "..." } }
\`\`\`

## Git Conventions

[Derive from: recent git log, CONTRIBUTING.md, or PR templates. If not determinable: ⚠️]

## Spec Mapping Rule

[Derive from: test describe block labels. If tests follow a pattern like `'JIRA-XXX | Scenario N — label'`, document it.]
```

---

#### `docs/arch/arch-diagrams.md`

**Derived from:** folder structure, `package.json`, ORM schema relations, route files.

```markdown
# Architecture: Diagrams

## C4 — Context (Level 1)

[Derive the system name from package.json `name` field or README title. Identify external users from the UserRole enum in the ORM schema.]

\`\`\`
[ASCII C4 context diagram — system + external users]
\`\`\`

## C4 — Container (Level 2)

[Derive containers from the actual folder structure:
- A `src/` with Express routes → REST API container
- A React/Vite `client/` or `frontend/` → SPA container
- A `prisma/schema.prisma` → Database container
]

\`\`\`
[ASCII C4 container diagram — frontend → API → DB]
\`\`\`

## Data Model — Core Entities

> The full data model is in [`arch-data-model.md`](arch-data-model.md).

[Quick-reference entity list and status flow derived from ORM schema.]
```

**How to derive the C4 diagrams:**
- System name: `package.json` `name` or first heading in `README.md`
- External actors: roles defined in the ORM schema enum (e.g., `UserRole`)
- Containers: presence of `src/api/` or `src/routes/` → REST API; presence of `client/` or `frontend/` or Vite config → SPA; presence of `prisma/schema.prisma` → PostgreSQL/relational DB
- Communication: if there is a `DATABASE_URL` in `.env.example` → API ↔ DB; if there is an `API_BASE_URL` in `.env.example` → Frontend ↔ API

---

#### `docs/arch/arch-data-model.md`

**Derived from:** ORM schema file (primary source). This is the richest document.

```markdown
# Architecture: Data Model

> Source of truth for spec writers...

## Entity Relationship Overview
[Derive from FK relations in prisma schema — @relation fields]

## Entities
[One section per model in the ORM schema. For each field: derive type, required/optional, constraints from schema modifiers (@unique, @default, ?, [])]

## Enums
[All enum blocks from the ORM schema, with a description column]

## State Machine
[If any field is an enum that looks like a lifecycle (DRAFT/PENDING/IN_PROGRESS/DONE/DELIVERED/CANCELLED), derive the state machine from:
  1. The enum values (order implies valid flow)
  2. Service function names (accept(), start(), complete(), deliver(), cancel())
  3. Test scenario labels that describe transitions]

## Cross-Cutting Patterns
[Derive from recurring field patterns:
  - `id UUID @default(uuid())` → UUID PKs pattern
  - `createdAt DateTime @default(now())` → audit fields pattern
  - `updatedAt DateTime @updatedAt` → auto-update pattern
  - `archivedAt DateTime?` → soft-delete pattern
  - `DECIMAL(10,2)` fields → monetary values pattern]

## Spec Writing Guide — Data Model Impact Section
[Include the 5-question checklist and the template — this section is static, copy it from the reference below]
```

**Spec writing guide section (copy verbatim into every `arch-data-model.md` generated):**

> Include the five questions and template from the reference `arch-data-model.md` in `docs/arch/`. These are static guidance — they do not change between projects.

---

### Step 4 — Generate `docs/features/`

Generate one `main.md` and one `feature-<domain>.md` per identified domain area.

---

#### `docs/features/main.md`

**Derived from:** route files (resource groups), ORM schema (entity names → feature areas), README, test file organization.

```markdown
# Product: [Name from package.json or README]

## Vision

[Derive from README first heading/paragraph. If no README: ⚠️ Fill in manually.]

## Users

[Derive from the UserRole enum in the ORM schema.]

| Role | Description |
|------|-------------|
| [Role] | [Derive from the role name and the routes it can access — infer from requireRole() calls in route files] |

## Feature Map

[Derive one feature per resource group in route files.
Group: /customers → Customer Management; /service-orders → Service & Maintenance Orders; /reports → Reporting & Analytics; /auth → Authentication]

| Feature | File | Priority | Status |
|---------|------|----------|--------|
| [Feature name] | `feature-<slug>.md` | [P1 for core CRUD, P2 for reporting] | [Planned / In Progress / Done — derive from test coverage] |

## SDD Flow

[Copy the SDD flow diagram from AGENTS.md if it exists, otherwise use the standard template:]

\`\`\`
Idea → specs/<TICKET>/ → feature-*.md → src/ → test/
\`\`\`

## MVP Scope

[Derive from: test file organization (which resource tests exist = what's been built), spec files in specs/ directory (which tickets have specs = what's planned), and route files (which routes exist = what's implemented)]
```

---

#### `docs/features/feature-<domain>.md`

One file per domain area. Derive the domain name from route files (e.g., `customer.routes.ts` → `feature-customer.md`).

```markdown
# Feature: [Domain Name]

## Goal

[Derive from: the service functions available (e.g., registerCustomer, findCustomerByPhone → "register and find customers"), the route paths, and any README section.]

## [Lifecycle / Workflow — if applicable]

[If this domain has a state machine (e.g., service_orders.status), include the status flow here. Derive from the ORM enum.]

## Epics

[Group the user stories into epics by functional area. Derive from:
  1. Test `describe` block labels — `'JIRA-XXX | Scenario N — label'` → extract the JIRA ID and label
  2. Spec files in `specs/JIRA-XXX/` — the user story inside spec-requirements.md
  3. Route file function names — registerCustomer, searchCustomer, archiveCustomer → separate stories]

### EP-[N] — [Epic Name]

[Short description of the epic's goal.]

**User Stories:**
- [JIRA-XXX]: [Story title — derive from spec-requirements.md User Story section, or from test describe label]

## Business Rules

[Derive from service files: every `if` condition that throws a `DomainError` is a business rule.
Example: `if (existing) throw new DomainError('DUPLICATE_PHONE', ...)` → "Phone number must be unique across all customers"]

## Out of Scope

[Derive from: spec-requirements.md "Out of Scope" sections. If no specs exist: ⚠️ Fill in manually.]
```

**How to derive epics from test files:**
- Scan all test `describe` labels
- Labels matching `'JIRA-NNN | Scenario N — ...'` → extract the JIRA ID
- Group JIRA IDs by the resource they test → that is an epic
- Check `specs/JIRA-NNN/spec-requirements.md` for the user story title

**How to derive business rules from service files:**
- Every `throw new DomainError(...)` or equivalent → one business rule
- Every `if (condition) throw` before a DB write → one business rule
- State machine validation logic → one rule per invalid transition

---

### Step 5 — Write the Files

Write all files using the Write tool. Do not use templates with unfilled placeholders in the final output — replace every `[placeholder]` with derived content, or replace with `> ⚠️ Not determinable from code — fill in manually` if the information genuinely cannot be derived.

File writing order (respects dependencies — data-model before diagrams, main.md before feature files):

1. `docs/arch/arch-data-model.md`
2. `docs/arch/arch-stack-tech.md`
3. `docs/arch/arch-conventions.md`
4. `docs/arch/arch-diagrams.md`
5. `docs/features/main.md`
6. `docs/features/feature-<domain>.md` (one per domain, alphabetical)

---

### Step 6 — Quality Check

Before reporting completion, verify:

**Coverage**
- [ ] Every route file has a corresponding `feature-<domain>.md`
- [ ] Every ORM model is documented in `arch-data-model.md`
- [ ] Every enum is listed with descriptions in `arch-data-model.md`
- [ ] `main.md` lists all features found in route files
- [ ] `arch-stack-tech.md` covers all dependencies in `package.json` that are non-trivial

**Accuracy**
- [ ] No placeholder text (`[placeholder]`, `TODO`, `...`) remains in any generated file
- [ ] All `⚠️` markers are for genuinely missing information — not for derivable content
- [ ] ORM field types and constraints match the actual schema file exactly
- [ ] Business rules in `feature-*.md` match actual `DomainError` throws in service files
- [ ] Role names match the actual enum values in the ORM schema

**Structure**
- [ ] All file names follow `kebab-case`
- [ ] `arch-diagrams.md` references `arch-data-model.md` instead of duplicating entity definitions
- [ ] `main.md` has a `Feature Map` table that lists all `feature-*.md` files

---

### Step 7 — Update Navigation

After generating all files, check if `AGENTS.md` exists in the project root. If it does, update the Navigation table to include any new files. If it does not exist, suggest creating it with the standard template from this workspace.

---

### Step 8 — Report Completion

```
## docs/ generated

### Files created / updated
docs/arch/
  arch-stack-tech.md     [created / updated]
  arch-conventions.md    [created / updated]
  arch-diagrams.md       [created / updated]
  arch-data-model.md     [created / updated]

docs/features/
  main.md                [created / updated]
  feature-<domain>.md    [created / updated — one line per file]

### Sources used
- [list the key files read: package.json, prisma schema, route files, test files]

### Manual review required (⚠️ items)
- [list any sections marked ⚠️ that need human input]

### Stats
- Entities documented: N
- Features identified: N
- Business rules derived: N
- Roles identified: N
```

---

### Step 9 — Offer Follow-up

> Would you like me to:
> - Run **`spec-creator`** for any of the user stories found in the feature files?
> - Generate the `AGENTS.md` agent instructions file for this project?
> - Add the `arch/` and `features/` docs to the `docs-creator` skill as a reference template?

---

## Derivation Reference — Key Mappings

| Source | Derived artifact |
|---|---|
| `package.json` `dependencies` | Stack table in `arch-stack-tech.md` |
| `package.json` `devDependencies` | Test framework + build tools in stack table |
| `package.json` `name` | Product name in `main.md` |
| `docker-compose.yml` / `DATABASE_URL` in `.env.example` | Database technology |
| `prisma/schema.prisma` models | Entities in `arch-data-model.md` |
| `prisma/schema.prisma` enums | Enums section + Role table in `main.md` |
| `prisma/schema.prisma` `@relation` | ERD and cardinality in `arch-data-model.md` |
| `prisma/schema.prisma` `@unique` / `@@unique` | Uniqueness constraints + business rules |
| `prisma/schema.prisma` nullable fields (`?`) | Required vs. optional in entity tables |
| Route file names (`customer.routes.ts`) | Domain names → `feature-customer.md` |
| Route middleware (`requireRole(...)`) | Auth NFRs + role permissions in `main.md` |
| Route HTTP methods + paths | API Contract section, C4 containers |
| Service `throw new DomainError(...)` | Business rules in `feature-*.md` |
| Service function names | Epic/story titles in `feature-*.md` |
| Test `describe` labels `'JIRA-XXX \| ...'` | User story IDs + titles in `feature-*.md` |
| `.env.example` variable names | Environment variables in `arch-stack-tech.md` |
| `.github/workflows/*.yml` | CI platform in stack table |
| `specs/JIRA-XXX/spec-requirements.md` | User story title, out-of-scope items, estimation |

## Notes on Non-Standard Projects

If the codebase does not follow the standard folder structure (`src/api/`, `src/services/`, `src/repositories/`):

- Look for files with route-like names: `*router*`, `*controller*`, `*handler*`, `*endpoint*`
- Look for files with service-like names: `*service*`, `*use-case*`, `*interactor*`, `*command*`
- Look for ORM files: `*entity*`, `*model*`, `*schema*`, any file with `@Entity`, `@Table`, `Model.define()`
- If the project has no ORM, look for SQL migration files to derive the data model
- If the project has no tests, skip test-derived content and mark as ⚠️
