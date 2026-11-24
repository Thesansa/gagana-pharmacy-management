# Gagana Pharmacy Management System
## Consolidated Architecture Decision Record (ADR) Register — ADR-001 to ADR-014

**Document status:** Approved architecture decisions  
**Project:** Gagana Pharmacy Inventory Management System  
**Architecture style:** Microservices  
**Backend platform:** .NET / ASP.NET Core  
**Database platform:** Microsoft SQL Server  

---

## 1. Purpose

This document is the consolidated record of all fourteen approved architecture decisions for the Gagana Pharmacy Management System, from ADR-001 through ADR-014. It is intended to be the single Markdown reference for the architecture decisions made so far.

The decisions establish the initial direction for service boundaries, data ownership, service communication, workflow coordination, security, authorization, and reference-data access. Detailed endpoints, schemas, message routing, retry policies, and implementation mechanics will be refined during service design.

## 2. Architecture context

The system is organized into seven independently deployable business services:

1. **Identity & Access Service** — user accounts, roles, employees, branches, and departments.
2. **Audit Service** — audit-event consumption and audit-log storage.
3. **Inventory Management Service** — branch inventory, stock settings, alerts, adjustments, and stock transactions.
4. **Medicine Catalog Service** — medicines, medicine batches, and batch price history.
5. **Stock Operations Service** — stock receipts, receipt items, sales, and sale items.
6. **Transfer Service** — stock-transfer requests, approvals, dispatch, and receipt workflow.
7. **Notification Service** — user notifications and notification read status.

The API Gateway is an infrastructure component, not an eighth business service.

Each service owns its data and database. Services must not directly read or write another service's tables.

## 3. Decision summary

| ADR | Decision | Approved approach |
|---|---|---|
| ADR-001 | User role assignment | One primary role per user account |
| ADR-002 | Employee branch assignment | One branch per employee |
| ADR-003 | Organization data ownership | Identity & Access owns Branch, Department, and Employee management |
| ADR-004 | Audit architecture | Audit is an independently deployed service |
| ADR-005 | API Gateway | YARP |
| ADR-006 | Message broker | RabbitMQ |
| ADR-007 | Audit communication | RabbitMQ with Transactional Outbox |
| ADR-008 | Sale and inventory coordination | Reservation and confirmation workflow |
| ADR-009 | Stock-transfer management | Explicit state machine with stock reservations |
| ADR-010 | Database deployment | Seven separate databases on one SQL Server instance |
| ADR-011 | Event contracts | Standardized, versioned JSON event envelopes |
| ADR-012 | Service authentication | User JWTs plus dedicated service identities |
| ADR-013 | Authorization | Role-based permissions with branch-level authorization |
| ADR-014 | Reference-data access | Hybrid: REST initially; local read models when needed |

---

## ADR-001: One Primary Role per User Account

**Status:** Accepted

### Context
The initial system defines four primary user roles: Admin, HR, Branch Manager, and Staff. The project needs a clear and manageable way to associate an account with its authorization role.

A user-role many-to-many relationship would allow multiple roles to be assigned to one account. That flexibility is not required by the current project scope and would introduce additional permission-combination and testing complexity.

### Decision
Each user account will have **one primary role**.

The `USER_ACCOUNT` entity will contain a `role_id` foreign key referencing `ROLE`. A separate user-role junction table is not required for the initial implementation.

### Rules
- Every active user account must have one primary role.
- The role must reference a valid record in `ROLE`.
- The four initial roles are Admin, HR, Branch Manager, and Staff.
- Role assignment and changes must be restricted to authorized account-management operations.
- Having one primary role does not remove the need for resource-level and branch-level authorization.

### Consequences
- The initial data model is simpler and easier to enforce.
- Authorization checks can begin with a single role claim.
- If multiple roles per account become a genuine requirement later, the data model and authorization model will need a deliberate migration.

---

## ADR-002: One Branch per Employee

**Status:** Accepted

### Context
Employees perform pharmacy operations in the context of a branch. The initial project scope assumes that an employee belongs to one branch, and branch-level authorization relies on that assignment.

Allowing multiple branch assignments at the beginning would add complexity to employee management, user claims, and authorization checks without being required by the current scope.

### Decision
Each employee will be assigned to **one branch at a time**.

The `EMPLOYEE` entity will contain a `branch_id` foreign key referencing `BRANCH`.

### Rules
- An employee's current branch assignment must reference a valid branch.
- Branch-scoped operations must be checked against the employee's assigned branch.
- A Branch Manager may perform managerial operations only within the branch to which they are assigned.
- A transfer approval must be checked against the source branch and the approving manager's branch assignment.
- If an employee changes branches, the employee's current assignment is updated through an authorized process. Historical records should retain the branch associated with the original business operation where needed.

### Consequences
- Branch authorization is straightforward for the initial system.
- Employee and user context can identify one current branch.
- Supporting employees who work across multiple branches would require a future design change, such as an employee-branch assignment entity.

---

## ADR-003: Identity & Access Owns Organization Data

**Status:** Accepted

### Context
The system needs to manage organizational reference data, including branches, departments, and employees. These records are closely related to user accounts, authentication, and authorization.

Placing the same entities under multiple services would create unclear ownership and duplicated management responsibilities.

### Decision
The **Identity & Access Service** owns the management and persistence of:
- `BRANCH`
- `DEPARTMENT`
- `EMPLOYEE`
- `ROLE`
- `USER_ACCOUNT`

### Rules
- Identity & Access is the authoritative owner of branch, department, employee, role, and user-account records.
- Other services may store external identifiers such as `branch_id` or user IDs as references in their own databases.
- Other services must not directly query or modify Identity & Access database tables.
- Other services obtain or validate organization data through Identity & Access APIs initially.
- If a future requirement justifies local projections, those projections may be maintained through defined events without transferring ownership.

### Consequences
- Organization and account management have one clear owner.
- Other services depend on Identity & Access contracts rather than its database schema.
- Identity & Access must provide the APIs needed for branch and employee validation.

---

## ADR-004: Audit as an Independently Deployed Service

**Status:** Accepted

### Context
The system must maintain a record of important user and system actions, including account changes, stock adjustments, sales, and transfer operations.

Audit information has cross-cutting value, but embedding audit-log storage and management independently inside every business service would create duplicated implementations and inconsistent audit records.

### Decision
Audit will be an **independently deployed Audit Service** with ownership of the `AUDIT_LOG` entity and its own database.

### Rules
- Audit Service owns audit-log persistence and retrieval.
- Business services produce audit information through defined communication contracts.
- Business services must not write directly to the Audit database.
- Audit Service is separate from ordinary stock transaction history. `STOCK_TRANSACTION` records inventory movements; `AUDIT_LOG` records who performed or caused an action and when.
- Audit delivery reliability is addressed by ADR-007.

### Consequences
- Audit behavior and storage are centralized.
- Business services do not need direct database access to audit records.
- Audit Service becomes an independently deployed component that must be monitored and integrated reliably.

---

## ADR-005: API Gateway Selection --- YARP

**Status:** Accepted

### Context

The system has multiple independently deployed backend services. The
frontend needs a consistent entry point for accessing them, without
needing to know each service's internal address and routing details.

### Decision

Use **YARP (Yet Another Reverse Proxy)** as the API Gateway.

### Rationale

-   YARP integrates with ASP.NET Core and fits the selected .NET backend
    platform.
-   It provides request routing and reverse-proxy capabilities.
-   ASP.NET Core middleware can be used for cross-cutting concerns such
    as authentication, rate limiting, and request handling.
-   It provides a clear entry point between the frontend and backend
    services.

### Responsibilities and boundaries

-   The frontend sends backend requests through the Gateway.
-   YARP routes requests to the appropriate service.
-   Gateway-level authentication and other cross-cutting middleware may
    be applied where appropriate.
-   Each backend service must still validate authentication and enforce
    its own authorization rules.
-   The Gateway does not replace service-level security or business
    validation.

### Consequences

-   A common frontend entry point is established.
-   Gateway route configuration must be maintained as services and
    endpoints evolve.
-   Gateway availability becomes important to incoming client traffic.

------------------------------------------------------------------------

## ADR-006: Message Broker Selection --- RabbitMQ

**Status:** Accepted

### Context

Services need to publish business events and allow other services to
react asynchronously. Direct REST communication remains useful when an
immediate response is required, but it should not be the only
communication mechanism.

### Decision

Use **RabbitMQ** as the message broker for asynchronous inter-service
communication.

### Rationale

-   RabbitMQ supports asynchronous message delivery and queue-based
    communication.
-   Consumers can process messages independently of the publishing
    service.
-   Messages can remain queued while a consumer is temporarily
    unavailable, subject to appropriate durable queue and message
    configuration.
-   It is a practical fit for the project's event-driven workflows and
    Docker-based development environment.

### Communication conventions

-   Use REST APIs when the caller needs an immediate response or
    synchronous validation.
-   Use RabbitMQ for asynchronous business events and decoupled
    notifications.
-   Do not replace every REST call with a message.
-   Define exchanges, queues, routing keys, retry behavior, dead-letter
    handling, and message durability deliberately.

### Consequences

-   Services can react to events without requiring the receiving service
    to be available at the exact time of publication.
-   RabbitMQ introduces additional infrastructure and operational
    configuration.
-   Consumers must handle retries and duplicate delivery safely.

------------------------------------------------------------------------

## ADR-007: Reliable Audit Communication --- RabbitMQ and Transactional Outbox

**Status:** Accepted

### Context

The Audit Service is independently deployed and owns the `AUDIT_LOG`
data. Other services must send audit information without directly
writing to the Audit database.

A simple fire-and-forget message publication can fail after a business
operation has committed, leaving an important action without its
corresponding audit event.

### Decision

Business services will send audit events through **RabbitMQ**, using the
**Transactional Outbox Pattern** for important auditable operations.

### Proposed workflow

1.  A business service performs an operation.
2.  In the same local database transaction, it saves the business change
    and a corresponding pending audit event in its own outbox table.
3.  A background publisher reads pending outbox records and publishes
    them to RabbitMQ.
4.  The publisher records successful publication so the event is not
    needlessly republished.
5.  The Audit Service consumes the event and persists the audit record
    in its own database.
6.  The Audit Service handles duplicate deliveries idempotently, using
    the event identifier.

### Rules

-   Business services must not write directly to the Audit database.
-   The Audit Service owns and manages its `AUDIT_LOG` table.
-   Outbox publication must support retries.
-   Audit-event consumption must be idempotent.
-   Audit events should contain the actor, action, affected entity
    reference, timestamp, and relevant correlation information.
-   Do not include passwords, secrets, access tokens, or unnecessary
    sensitive data in audit events.

### Consequences

-   The business change and the creation of its pending audit event are
    committed atomically within the business service's database.
-   Audit delivery is asynchronous, so an audit record may appear
    shortly after the business operation.
-   Outbox tables, background publishers, retries, and consumer
    idempotency must be implemented and tested.

------------------------------------------------------------------------

## ADR-008: Sale and Inventory Coordination --- Reservation and Confirmation

**Status:** Accepted

### Context

The Stock Operations Service owns sales, while the Inventory Management
Service owns stock quantities and stock transactions. The services have
separate databases, so a single ordinary SQL transaction cannot
atomically update both services' data.

Creating a completed sale first and only then attempting to deduct stock
could leave a completed sale without a corresponding stock deduction.

### Decision

Use a **reservation-and-confirmation workflow** between Stock Operations
and Inventory Management.

### Proposed workflow

1.  Stock Operations receives a sale request with the selected medicine
    batches and quantities.
2.  Stock Operations requests an inventory reservation.
3.  Inventory validates the batch, expiry status, branch, and available
    quantity.
4.  Inventory atomically reserves the requested quantity. Concurrent
    requests must not be able to reserve the same units.
5.  Stock Operations persists the sale through its own transaction and
    tracks its processing state.
6.  After the sale is committed, the workflow confirms the reservation
    with Inventory.
7.  Inventory finalizes the reservation and records the stock movement
    in `STOCK_TRANSACTION`.
8.  If the sale cannot be completed, the reservation is released.
9.  Retries, idempotency, and reconciliation handle delayed or
    interrupted workflow steps.

### Inventory quantity semantics

For this workflow, `available_quantity` represents stock available for
new sales or reservations, while `reserved_quantity` represents stock
held for in-progress operations.

Example: reserving five units from fifty changes available quantity from
50 to 45 and reserved quantity from 0 to 5. On confirmation, reserved
quantity becomes 0; the five units are not deducted from available
quantity a second time.

### Rules

-   A sale must not be marked completed unless stock reservation has
    succeeded and the workflow reaches its required confirmation point.
-   Inventory is the authority for stock availability and reservation.
-   Reservation, confirmation, and release operations must be
    idempotent.
-   Expired batches cannot be sold.
-   Stock updates must be protected against concurrent overselling.
-   Unfinished reservations need an expiry or reconciliation mechanism.

### Consequences

-   The workflow is more reliable than creating a sale and
    asynchronously attempting an unprotected deduction.
-   The implementation needs explicit sale/reservation states and
    failure handling.
-   This is a lightweight Saga-style workflow; it does not require a
    distributed database transaction coordinator.

------------------------------------------------------------------------

## ADR-009: Stock Transfer Lifecycle --- State Machine and Reservations

**Status:** Accepted

### Context

The Transfer Service owns transfer requests and their workflow.
Inventory Management owns stock quantities and stock movements. A
transfer must prevent requested stock from being sold or allocated to
another operation before dispatch.

### Decision

Use an **explicit transfer state machine with inventory reservations**.

### Proposed lifecycle

The normal path is:

`REQUESTED → APPROVED → RESERVED → DISPATCHED → RECEIVED`

Additional outcomes include:

-   `REJECTED` --- the source Branch Manager rejects the request.
-   `CANCELLED` --- an eligible user cancels the transfer before
    dispatch, subject to the defined rules.
-   `PARTIALLY_RECEIVED` --- the destination accepts fewer units than
    were dispatched.

### Proposed workflow

1.  Staff submits a transfer request.
2.  The source Branch Manager approves or rejects it.
3.  After approval, Transfer requests Inventory to reserve the requested
    quantities at the source branch.
4.  Inventory validates and reserves available stock.
5.  At dispatch, Inventory converts the source reservation into an
    outgoing stock movement and records the transaction.
6.  The destination branch records the received and accepted quantities.
7.  Inventory adds only accepted quantities to destination stock.
8.  Any missing, damaged, or otherwise unaccepted quantities are
    recorded as discrepancies with reasons.

### Rules

-   Source and destination branches must differ.
-   The approving Branch Manager must be assigned to the source branch.
-   A Branch Manager may approve only transfers originating from their
    assigned branch.
-   Inventory owns and changes stock quantities; Transfer owns the
    transfer workflow.
-   Transfer must not directly update Inventory tables.
-   Stock movements must be recorded in `STOCK_TRANSACTION`.
-   Cancellation and reservation release are permitted only in valid
    states.
-   Partial dispatch and receipt behavior must be explicitly validated
    and tested.

### Consequences

-   Transfer states and allowed transitions must be enforced by the
    Transfer Service.
-   Inventory reservation and movement operations must be idempotent.
-   The system can distinguish requested, reserved, dispatched, and
    accepted quantities.

------------------------------------------------------------------------

## ADR-010: Database Deployment --- Seven Databases on One SQL Server Instance

**Status:** Accepted

### Context

Each microservice must own its data. Running a separate SQL Server
instance for every service would add unnecessary infrastructure overhead
for this project.

### Decision

Use **seven separate databases on one Microsoft SQL Server instance**.

The initial database allocation is:

  Service                Database
  ---------------------- -----------------
  Identity & Access      Identity DB
  Audit                  Audit DB
  Inventory Management   Inventory DB
  Medicine Catalog       Catalog DB
  Stock Operations       Operations DB
  Transfer               Transfer DB
  Notification           Notification DB

### Rules

-   Each service owns its database, schema, and migrations.
-   Services must not directly query or modify another service's tables.
-   Do not create cross-service foreign keys.
-   Store external identifiers as references in the owning service's
    database where required.
-   Use APIs or events to obtain or synchronize data owned by another
    service.
-   Use separate database credentials and restrict permissions so each
    service can access only its own database wherever practical.

### Consequences

-   Logical database ownership is maintained without running seven
    database-server instances.
-   One SQL Server instance remains a shared infrastructure dependency.
-   Local development can use one SQL Server instance/container
    alongside RabbitMQ.

------------------------------------------------------------------------

## ADR-011: Event Contract Standardization --- Versioned JSON Envelopes

**Status:** Accepted

### Context

Services exchange events through RabbitMQ. Without a common structure,
event identification, tracing, duplicate detection, and future contract
changes can become inconsistent.

### Decision

All inter-service business events will use **standardized, versioned
JSON event envelopes**.

### Standard envelope

``` json
{
  "eventId": "unique-event-id",
  "eventType": "LowStockDetected",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T10:30:00Z",
  "producer": "inventory-service",
  "correlationId": "operation-id",
  "causationId": "previous-event-id",
  "payload": {
    "branchId": 3,
    "medicineId": 105,
    "availableQuantity": 8,
    "reorderLevel": 15
  }
}
```

### Field conventions

  -----------------------------------------------------------------------
  Field                               Purpose
  ----------------------------------- -----------------------------------
  `eventId`                           Unique identifier used for
                                      duplicate detection

  `eventType`                         Business event name

  `eventVersion`                      Version of the event contract

  `occurredAt`                        Event creation timestamp in UTC

  `producer`                          Service that published the event

  `correlationId`                     Identifier connecting events in one
                                      workflow

  `causationId`                       Identifier of the event or action
                                      that caused this event

  `payload`                           Event-specific business data
  -----------------------------------------------------------------------

### Rules

-   JSON is the initial message format.
-   Timestamps use UTC and ISO 8601 representation.
-   Every event has a unique identifier and explicit version.
-   Consumers must be idempotent.
-   Event contracts must be documented and versioned.
-   Payloads should contain only the data consumers need.
-   Never publish passwords, JWT secrets, access tokens, or unnecessary
    personal information.
-   A schema registry is not required for the initial project scope;
    versioned DTOs and shared contract documentation are sufficient.

### Consequences

-   Event producers and consumers share predictable message structures.
-   Contract changes require versioning and compatibility consideration.
-   Event IDs and correlation IDs support tracing and reliable
    processing.

------------------------------------------------------------------------

## ADR-012: Service Authentication --- User JWTs and Dedicated Service Identities

**Status:** Accepted

### Context

The system needs to authenticate both end users and backend services.
Trusting only the internal network or treating every service as
interchangeable would create weak security boundaries.

### Decision

Use **JWT bearer authentication for users and dedicated service
identities for service-to-service authentication**.

### User authentication

-   Identity & Access issues user JWTs after successful authentication.
-   Backend services validate token signature, issuer, audience, and
    expiry.
-   Services use validated identity claims as input to their own
    authorization checks.
-   The Gateway may validate incoming tokens, but services must not rely
    on the Gateway as their only security boundary.

### Service authentication

-   Each service has its own identity, such as `inventory-service` or
    `transfer-service`.
-   Internal calls authenticate using service-specific tokens or
    credentials.
-   A service identity should receive only the permissions needed for
    its responsibilities.
-   Credentials must be stored outside source control and supplied
    through secure configuration.
-   A service must not trust arbitrary identity or role headers supplied
    by a browser.

### Consequences

-   User and service identities are distinguishable.
-   Credentials can be scoped and rotated independently.
-   Service-to-service authentication and authorization configuration
    must be implemented and tested.
-   A lightweight signed service-token approach may be used initially; a
    full OAuth2 authorization server is not required unless project
    needs justify it.

------------------------------------------------------------------------

## ADR-013: Authorization --- RBAC with Branch-Level Restrictions

**Status:** Accepted

### Context

The system has four primary roles: Admin, HR, Branch Manager, and Staff.
Each user has one primary role, and each employee belongs to one branch.

Role checks alone are insufficient because a Branch Manager must not
automatically access or approve operations belonging to another branch.

### Decision

Use **role-based permissions combined with branch-level authorization**.

### Authorization principles

-   Role determines which categories of actions a user may perform.
-   Branch assignment determines which branch-owned resources the user
    may access or modify.
-   Services enforce authorization on the backend for both read and
    write operations.
-   The backend must derive branch scope from trusted, validated
    identity information or verify it with Identity & Access. It must
    not blindly trust a branch ID supplied by the frontend.
-   Each service enforces the permissions relevant to its own business
    operations.

### Initial role boundaries

  -----------------------------------------------------------------------
  Role                                Initial responsibility boundary
  ----------------------------------- -----------------------------------
  Admin                               System-wide administration and
                                      configuration

  HR                                  Employee records and account
                                      lifecycle; not inventory approvals

  Branch Manager                      Managerial operations within the
                                      manager's assigned branch

  Staff                               Permitted day-to-day operations
                                      within the employee's assigned
                                      branch
  -----------------------------------------------------------------------

### Examples

-   A Branch Manager assigned to Colombo may approve an eligible
    transfer only when Colombo is the source branch.
-   A Branch Manager assigned to Colombo must not approve a transfer
    originating from Negombo.
-   Staff can perform permitted operations only within their authorized
    branch scope.
-   HR can manage employee records across branches but does not thereby
    gain inventory approval permissions.
-   Admin's system-wide administrative access remains distinct from HR's
    employee-management responsibilities.

### Consequences

-   Authorization checks must consider both role and resource scope.
-   A documented permission matrix should guide implementation and
    testing.
-   Branch-scoped access must be tested for both allowed and denied
    cases.

------------------------------------------------------------------------

## ADR-014: Reference-Data Access --- Hybrid REST and Local Read Models

**Status:** Accepted

### Context

Services own separate databases but need information owned by other
services. Examples include validating medicine batches, checking branch
references, and identifying notification recipients.

Direct database access across service boundaries is prohibited. Fully
duplicating reference data from the beginning would add synchronization
complexity before its value is established.

### Decision

Use a **hybrid approach: REST APIs initially, with local read models
introduced when a clear performance or availability requirement
exists**.

### Initial approach

-   Use the owning service's REST API when a service needs immediate
    validation or authoritative reference information.
-   Keep external identifiers in the consuming service's database where
    needed, without cross-service foreign keys.
-   Do not directly query another service's database.
-   Use bulk API operations where repeated individual calls would be
    inefficient.

### When to introduce local read models

Consider an event-fed local read model when:

-   A service repeatedly needs the same reference data at high volume.
-   Repeated synchronous API calls create a measurable performance
    problem.
-   A consumer needs to continue serving suitable read operations during
    temporary unavailability of the owning service.
-   The team can implement and operate synchronization, idempotency, and
    reconciliation reliably.

### Consequences

-   Initial implementation remains simpler and preserves clear data
    ownership.
-   Local projections can be added selectively as actual needs emerge.
-   Read models may be eventually consistent and must not be treated as
    the authoritative source for critical validation unless the workflow
    explicitly accounts for that limitation.

------------------------------------------------------------------------

## 4. Cross-decision implementation principles

The following principles apply across the accepted decisions:

1.  **Service ownership:** Each service owns its business logic, data,
    database, and migrations.
2.  **No shared-table access:** Services communicate through APIs and
    events, not another service's tables.
3.  **Synchronous versus asynchronous communication:** Use REST when an
    immediate response is needed; use RabbitMQ for asynchronous events.
4.  **Reliable event publication:** Use the Transactional Outbox Pattern
    for important operations where losing an event would create an
    integrity or audit problem.
5.  **Idempotency:** Message consumers and workflow operations must
    tolerate retries and duplicate delivery.
6.  **Workflow integrity:** Sales and transfers must use explicit
    reservation, confirmation, release, and state-transition rules.
7.  **Defense in depth:** Gateway checks do not replace service-level
    authentication and authorization.
8.  **Branch isolation:** Branch-scoped access must be enforced
    server-side.
9.  **Contract discipline:** API and event contracts must be documented
    and versioned.
10. **Avoid premature complexity:** Introduce read models and additional
    infrastructure only when a concrete requirement justifies them.

## 5. Implementation follow-up

These decisions establish the architecture direction; they do not by
themselves define every endpoint, message, database column, retry
policy, or state transition. Those details should be specified during
service design.

The next planning activities are:

1.  Finalize implementation allocation and service ownership among team
    members.
2.  Define the repository and .NET solution structure.
3.  Define each service's project structure, API boundaries, and
    database migrations.
4.  Specify shared API conventions and versioned event contracts.
5.  Design the sale and transfer state machines in implementation-level
    detail.
6.  Define the permission matrix and service-to-service access matrix.
7.  Plan local infrastructure using Docker Compose for SQL Server,
    RabbitMQ, Gateway, and services.

------------------------------------------------------------------------

## 4. Cross-decision implementation principles

1. Each service owns its business logic, data, database, and migrations.
2. Services communicate through APIs and events, not another service's tables.
3. Use REST when an immediate response is needed; use RabbitMQ for asynchronous events.
4. Use the Transactional Outbox Pattern for important events where losing publication would create an integrity or audit problem.
5. Consumers and workflow operations must handle retries and duplicate delivery safely.
6. Sales and transfers use explicit reservation, confirmation, release, and state-transition rules.
7. Gateway checks do not replace service-level authentication and authorization.
8. Branch-scoped access is enforced on the backend.
9. API and event contracts are documented and versioned.
10. Add read models and infrastructure complexity only when a concrete requirement justifies them.

## 5. Implementation follow-up

1. Finalize implementation allocation and service ownership among team members.
2. Define the repository and .NET solution structure.
3. Define each service's project structure, API boundaries, and database migrations.
4. Specify shared API conventions and versioned event contracts.
5. Design sale and transfer state machines in implementation-level detail.
6. Define the permission matrix and service-to-service access matrix.
7. Plan local infrastructure using Docker Compose for SQL Server, RabbitMQ, the Gateway, and services.

**Document note:** This register contains the fourteen architecture decisions ADR-001 through ADR-014. These decisions form the approved architecture baseline; detailed implementation contracts and workflows will be specified during the next design phase.
