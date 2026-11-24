# ADR-001: One Primary Role per User Account

- **Status:** Accepted
- **Date:** 24 November 2026

## Context
The initial user and authorization model needs a clear rule for role assignment.

## Decision
Each user account has one primary role. The initial roles are Admin, HR, Branch Manager and Staff.

## Consequences
- `USER_ACCOUNT.role_id` references `ROLE.role_id`.
- A separate `USER_ROLE` junction table is not required for the current model.
- If multiple simultaneous roles become a requirement, revisit the model through a new architecture decision.
