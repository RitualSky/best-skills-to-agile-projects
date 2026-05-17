# Feature: Customer Management

## Goal

Allow receptionists to register, search, and view the service history of customers who bring devices to the workshop.

## Epics

### EP-01 — Customer Registration
Register a new customer with contact information and device details.

**User Stories:**
- JIRA-123: Register a new customer with name, phone, and email
- JIRA-127: Prevent duplicate registration by phone number
- JIRA-128: Edit existing customer information

### EP-02 — Customer Search
Find a customer quickly at the moment they arrive at the shop.

**User Stories:**
- JIRA-129: Search customer by name or phone number
- JIRA-130: View customer profile with full service history

## Business Rules

- Phone number is the unique identifier for a customer.
- Email is optional but required to send status notifications.
- A customer cannot be deleted if they have open service orders.
- Customer records are soft-deleted (archived) to preserve order history.

## Out of Scope

- Customer loyalty/points program
- Online self-service portal for customers
- Multi-location customer sharing
