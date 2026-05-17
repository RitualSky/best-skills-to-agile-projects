# Architecture: Technology Stack

## Guiding Principles

- **Spec first:** No code is written without a spec in `docs/specs/`.
- **Simplicity over cleverness:** Use boring, proven technology.
- **Testability:** Every layer must be independently testable.

## Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | React 18 + TypeScript | Component model fits the form-heavy UI; strong typing reduces runtime errors |
| Backend | Node.js 20 + Express | Familiar ecosystem; lightweight enough for this domain |
| Database | PostgreSQL 16 | Relational model fits customers ↔ orders; ACID guarantees for status transitions |
| ORM | Prisma | Type-safe queries; migration-as-code aligns with spec-driven approach |
| Auth | JWT + bcrypt | Stateless, simple to implement for a single-shop deployment |
| Testing (unit) | Vitest | Fast, native ESM support |
| Testing (e2e) | Playwright | BDD scenarios map directly to page-object tests |
| CI | GitHub Actions | Free for public repos; matrix builds for node versions |
| Hosting | Railway (MVP) | Zero-config PostgreSQL + Node deployment |

## Folder Structure

```
src/
  api/          Express routes (one file per resource)
  services/     Business logic (pure functions, no HTTP concerns)
  repositories/ Database access via Prisma
  models/       TypeScript types and Zod schemas
  utils/        Shared helpers

test/
  unit/         Vitest — services and repositories
  integration/  Vitest + test DB — full service layer
  e2e/          Playwright — BDD scenarios from spec-bdd.md
```

## Decision Log

| Decision | Chosen | Rejected | Reason |
|----------|--------|----------|--------|
| ORM | Prisma | TypeORM | Prisma's migration CLI is cleaner; type inference is superior |
| Testing framework | Vitest | Jest | Faster cold start; ESM-native |
| Database | PostgreSQL | MongoDB | Order lifecycle state machine benefits from relational constraints |
