# ADR-003: Identity & Access Owns Organization Records

- **Status:** Accepted
- **Date:** 24 November 2026

## Context
The architecture needs an owner for branch, department and employee information used by multiple services.

## Decision
Identity & Access owns `BRANCH`, `DEPARTMENT`, `EMPLOYEE`, `ROLE` and `USER_ACCOUNT`.

## Consequences
- Other services store branch and user identifiers as external references.
- Other services obtain authoritative organization and identity information through APIs or trusted events.
- No service directly queries or modifies the Identity database.
