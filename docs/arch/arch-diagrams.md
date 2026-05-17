# Architecture: Diagrams

## C4 — Context (Level 1)

```
┌─────────────────────────────────────────────────────────┐
│                    TechShop System                       │
│                                                          │
│  ┌──────────────┐    ┌───────────────────────────────┐  │
│  │  React SPA   │◄──►│  Express REST API              │  │
│  │  (Browser)   │    │  (Node.js)                    │  │
│  └──────────────┘    └───────────────┬───────────────┘  │
│                                      │                   │
│                             ┌────────▼────────┐         │
│                             │  PostgreSQL DB   │         │
│                             └─────────────────┘         │
└─────────────────────────────────────────────────────────┘

External users: Receptionist, Technician, Manager
```

## C4 — Container (Level 2)

```
[Receptionist / Technician / Manager]
          │
          ▼ HTTPS
  ┌───────────────┐
  │  React SPA    │  Vite build, served as static files
  └───────┬───────┘
          │ REST JSON
          ▼
  ┌───────────────┐
  │  Express API  │  Routes → Services → Repositories
  └───────┬───────┘
          │ Prisma ORM
          ▼
  ┌───────────────┐
  │  PostgreSQL   │  customers, service_orders, users, order_logs
  └───────────────┘
```

## Data Model — Core Entities

> The full data model (entities, fields, constraints, enums, state machine, business rules, and Zod validation references) has been moved to [`arch-data-model.md`](arch-data-model.md).

**Quick reference — entity list:**

| Entity | Purpose |
|--------|---------|
| `customers` | Contact information of device owners |
| `service_orders` | Repair/maintenance jobs — lifecycle tracked via status |
| `order_logs` | Immutable audit trail of every status transition |
| `users` | System users (Receptionist, Technician, Manager) |

**Quick reference — status flow:**
```
DRAFT → PENDING → IN_PROGRESS ⇄ WAITING_PARTS → DONE → DELIVERED
                                                       ↘ CANCELLED (from any non-terminal state)
```

See [`arch-data-model.md`](arch-data-model.md) for the complete state machine, valid transitions, and role permissions.
