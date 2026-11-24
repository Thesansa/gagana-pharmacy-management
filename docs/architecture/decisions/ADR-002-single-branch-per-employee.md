# ADR-002: One Branch per Employee

- **Status:** Accepted
- **Date:** 24 November 2026

## Context
Branch-scoped access and employee ownership need a clear initial assignment rule.

## Decision
Each employee belongs to one branch.

## Consequences
- `EMPLOYEE.branch_id` references `BRANCH.branch_id`.
- Branch-scoped authorization can use the employee's assigned branch.
- If employees later need multiple branch assignments, consider an employee-branch association model.
