# AGENTS.md — Agent Instructions

## Project Overview

This repository demonstrates **Spec Driven Development (SDD)** applied to an agile project.

**Domain:** Customer Technical Service Workshop — a system that manages customers, service/maintenance orders, and business reporting for a small-to-medium repair shop.

**SDD Workflow in this project:**

```
1. docs/features/      → Define WHAT to build (feature scope, epics, user stories)
2. docs/specs/<TICKET>/spec-requirements.md  → Formal requirements for one story
3. docs/specs/<TICKET>/spec-bdd.md           → HOW to verify it (BDD scenarios)
4. src/                → Code that satisfies the spec
5. test/               → Automated tests derived 1:1 from BDD scenarios
```

**Skill pipeline (use skills in this order for a new ticket):**

```
spec-creator  →  bdd-creator  →  apply-spec
  JIRA/text       spec-bdd        code + tests
  ↓                ↓                ↓
spec-requirements  Gherkin      src/ + test/
```

The spec is the source of truth. Code exists to satisfy the spec, not the other way around.

## Navigation

| Path | Purpose |
|------|---------|
| `docs/features/main.md` | Full product vision and feature index |
| `docs/features/feature-*.md` | Scope and stories per feature area |
| `docs/arch/` | Architecture decisions, stack, conventions |
| `docs/arch/arch-data-model.md` | **Spec reference** — entities, field constraints, enums, state machine, business rules, Zod validation, spec writing guide |
| `docs/specs/<JIRA-ID>/` | Per-ticket specification (requirements + BDD) |
| `src/` | Application source code |
| `test/` | Automated tests (unit, integration, e2e) |
| `.agents/skills/` | Claude Code skills for this project |

## Available Skills

- **docs-creator** — Reads the entire codebase (routes, services, ORM schema, tests, config) and generates or regenerates `docs/arch/` and `docs/features/` with all architecture and feature documentation. Safe to run on an existing `docs/` folder.
- **spec-creator** — Creates `spec-requirements.md` for a ticket. Accepts a JIRA ticket ID (fetches via REST API) or a free-text description. Covers all spec sections: User Story, FRs, NFRs, Data Model Impact, API Contract, Out of Scope, DoD, and Estimation.
- **enrich-us** — Enriches a rough user story into a full agile artifact with acceptance criteria, DoD, NFRs, estimation, and test scenarios.
- **bdd-creator** — Generates a complete `spec-bdd.md` from a `spec-requirements.md`. Covers happy paths, validation errors, business rules, auth scenarios, and boundary values. Produces a Traceability Matrix and a Coverage Summary.
- **apply-spec** — Implements in code a full JIRA ticket from its spec files. Produces model (Zod + types), repository (Prisma), service (business logic), route (Express), Prisma migration, and all tests (unit, integration, e2e) — one test per BDD scenario.

## Agent Behavior Guidelines

- Always read `docs/features/main.md` first to understand the full product context.
- Before implementing any story, read its `docs/specs/<JIRA-ID>/` files. Never code without a spec.
- BDD scenarios in `spec-bdd.md` map 1:1 to test files in `test/`. Do not skip a scenario.
- If a requirement is ambiguous, stop and ask. Do not assume.
- Keep `src/` and `test/` mirroring each other in folder structure.
