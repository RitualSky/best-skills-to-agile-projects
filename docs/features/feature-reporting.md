# Feature: Reporting & Analytics

## Goal

Give the manager visibility into workshop operations through pre-built reports and a summary dashboard — no SQL or exports required.

## Epics

### EP-06 — Operations Dashboard
A real-time view of current workshop activity.

**User Stories:**
- JIRA-126: Orders by status widget (count + list)
- JIRA-137: Technician workload — open orders per technician
- JIRA-138: Orders overdue (past estimated delivery date)

### EP-07 — Revenue Reports
Track income over time and identify top services.

**User Stories:**
- JIRA-139: Monthly revenue summary (delivered orders)
- JIRA-140: Revenue breakdown by service type
- JIRA-141: Average ticket value over a date range

### EP-08 — Customer Reports
Understand customer behaviour and retention.

**User Stories:**
- JIRA-142: New customers per month
- JIRA-143: Returning customers (more than one order in 12 months)
- JIRA-144: Customers with no activity in 6+ months (churn risk)

## Key Metrics (KPIs)

| KPI | Definition |
|-----|-----------|
| Avg. resolution time | Days from PENDING to DONE |
| First-visit fix rate | Orders resolved without WAITING_PARTS status |
| Monthly recurring customers | Customers with 2+ orders in the last 30 days |
| Revenue per technician | Total final price of DELIVERED orders per technician |

## Out of Scope (MVP)

- Custom report builder
- Data export to Excel
- External BI tool integration (Tableau, Power BI)
