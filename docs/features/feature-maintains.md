# Feature: Service & Maintenance Orders

## Goal

Track the full lifecycle of a device repair or maintenance job — from the moment the customer drops off the device to the moment it is delivered back.

## Order Lifecycle

```
DRAFT → PENDING → IN_PROGRESS → WAITING_PARTS → DONE → DELIVERED
                                                      ↘ CANCELLED
```

| Status | Who sets it | Meaning |
|--------|-------------|---------|
| DRAFT | System | Order created but not yet accepted |
| PENDING | Receptionist | Accepted, waiting for technician assignment |
| IN_PROGRESS | Technician | Actively being worked on |
| WAITING_PARTS | Technician | Blocked — needs a part or external repair |
| DONE | Technician | Work complete, ready for pickup |
| DELIVERED | Receptionist | Device handed back to customer |
| CANCELLED | Any | Order voided before completion |

## Epics

### EP-03 — Order Creation
Open a new service order for an existing customer.

**User Stories:**
- JIRA-124: Create a service order linked to a customer
- JIRA-131: Attach device details (brand, model, serial number, reported issue)
- JIRA-132: Set estimated delivery date and quoted price

### EP-04 — Order Tracking
Allow technicians and receptionists to update and monitor order progress.

**User Stories:**
- JIRA-125: Transition order status with a comment log
- JIRA-133: Assign order to a specific technician
- JIRA-134: Log internal work notes per status change

### EP-05 — Order Delivery
Close the order when the customer picks up the device.

**User Stories:**
- JIRA-135: Mark order as DELIVERED and record final price
- JIRA-136: Generate a PDF receipt for the customer

## Business Rules

- An order must be linked to an existing customer before it can be saved.
- Status transitions must follow the defined lifecycle — no skipping states.
- A DONE or DELIVERED order cannot be edited, only viewed.
- Estimated price is mandatory; final price is set at delivery.
