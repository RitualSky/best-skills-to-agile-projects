# Product: TechShop — Technical Service Workshop Manager

## Vision

A lightweight web application that allows a small technical repair shop to manage its customers, track service/maintenance orders end-to-end, and generate operational reports — replacing manual spreadsheets and paper-based workflows.

## Users

| Role | Description |
|------|-------------|
| **Receptionist** | Registers customers, opens service orders, communicates status to clients |
| **Technician** | Updates order progress, logs work done, marks orders as complete |
| **Manager** | Reviews reports, monitors KPIs, manages pricing and technician workload |

## Feature Map

| Feature | File | Priority | Status |
|---------|------|----------|--------|
| Customer Management | `feature-customer.md` | P1 | Planned |
| Service & Maintenance Orders | `feature-maintains.md` | P1 | Planned |
| Reporting & Analytics | `feature-reporting.md` | P2 | Planned |

## SDD Flow

```
Idea → specs/<TICKET>/ → feature-*.md → src/ → test/
```

Each ticket in `specs/` corresponds to one user story. The spec files define the contract; the code fulfills it.

## MVP Scope (Sprint 1–3)

1. Register and search customers (JIRA-123)
2. Open a new service order linked to a customer (JIRA-124)
3. Update order status: Pending → In Progress → Done (JIRA-125)
4. Basic dashboard: orders by status (JIRA-126)
