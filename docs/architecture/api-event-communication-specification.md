# Gagana Pharmacy Management System
# API & Event Communication Specification — Version 1

**Status:** Draft for team review  
**Version:** 1.0  
**Last updated:** 24 November 2026  
**Related documents:**
- `microservice-architecture-v1.md`
- `service-responsibility-entity-ownership.md`

---

## 1. Purpose and status

This document defines proposed communication contracts between the React frontend, API Gateway and seven backend services. It identifies service ownership, proposed REST endpoints, event responsibilities, workflows, security and reliability expectations.

**Important:** Endpoints and payloads in this version are proposed contracts for discussion. They are not implemented APIs. Service leads must review them before coding and align exact fields with the final database design.

## 2. Communication principles

### REST
Use synchronous REST when the caller needs an immediate response, such as login, medicine search, batch lookup, inventory retrieval, receipt creation, sale processing, transfer requests or marking a notification as read.

### Events
Use asynchronous events when a completed action must be communicated to other services independently, such as receipt approval, sale processing, transfer dispatch, inventory alert detection or audit submission.

### Ownership
- A service modifies only its own database.
- A service must not directly query another service's database.
- Cross-service references use stable identifiers.
- Consumers must tolerate duplicate event delivery.

### API versioning
Use a version prefix, for example `/api/v1/medicines`. Breaking changes require a new version or documented migration strategy.

## 3. API Gateway routing

The Gateway exposes one base URL to React and routes requests to the relevant service.

| Gateway route prefix | Destination |
|---|---|
| `/api/v1/auth/**` | Identity & Access |
| `/api/v1/users/**` | Identity & Access |
| `/api/v1/branches/**` | Identity & Access |
| `/api/v1/employees/**` | Identity & Access |
| `/api/v1/audit/**` | Audit |
| `/api/v1/inventory/**` | Inventory Management |
| `/api/v1/adjustments/**` | Inventory Management |
| `/api/v1/alerts/**` | Inventory Management |
| `/api/v1/medicines/**` | Medicine Catalog |
| `/api/v1/batches/**` | Medicine Catalog |
| `/api/v1/receipts/**` | Stock Operations |
| `/api/v1/sales/**` | Stock Operations |
| `/api/v1/transfers/**` | Transfer |
| `/api/v1/notifications/**` | Notification |

These are proposed public route prefixes. Internal service routes may differ.

## 4. Identity & Access API

**Owner:** Identity & Access Service

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| POST | `/api/v1/auth/login` | Authenticate and issue access token | Public |
| POST | `/api/v1/auth/refresh` | Refresh access token, if implemented | Refresh-token flow |
| POST | `/api/v1/auth/logout` | Revoke refresh-token session, if implemented | Authenticated |
| GET | `/api/v1/users/me` | Current user's identity and claims | Authenticated |
| GET | `/api/v1/users` | List accounts | Admin / HR as authorized |
| POST | `/api/v1/users` | Create account | Admin / HR as authorized |
| GET | `/api/v1/roles` | List roles | Policy-controlled |
| GET | `/api/v1/branches` | List branches | Authenticated |
| POST | `/api/v1/branches` | Create branch | Admin |
| GET | `/api/v1/employees` | List employees | HR / Admin as authorized |
| POST | `/api/v1/employees` | Create employee | HR / Admin as authorized |
| PATCH | `/api/v1/employees/{employeeId}` | Update employee | HR / Admin as authorized |

The detailed permission matrix must be documented separately. The Gateway must not be the only enforcement point.

## 5. Medicine Catalog API

**Owner:** Medicine Catalog Service  
**Lead:** Nethmini

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/medicines` | Search/list medicines |
| GET | `/api/v1/medicines/{medicineId}` | Retrieve medicine |
| POST | `/api/v1/medicines` | Register medicine |
| PUT | `/api/v1/medicines/{medicineId}` | Update medicine |
| GET | `/api/v1/medicines/{medicineId}/batches` | List medicine batches |
| GET | `/api/v1/batches/{batchId}` | Retrieve batch |
| POST | `/api/v1/medicines/{medicineId}/batches` | Register batch |
| PATCH | `/api/v1/batches/{batchId}` | Update permitted batch details |
| GET | `/api/v1/batches/{batchId}/price-history` | Retrieve price history |

Inventory, Operations and Transfer may need Catalog data. They should use internal lookup endpoints or trusted read models. Other services must not write Catalog records.

## 6. Inventory Management API

**Owner:** Inventory Management Service  
**Lead:** Project technical lead

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/inventory` | Retrieve inventory within authorized scope |
| GET | `/api/v1/inventory/branches/{branchId}` | Retrieve branch stock |
| GET | `/api/v1/inventory/batches/{batchId}` | Retrieve stock for a batch |
| POST | `/api/v1/inventory/reservations` | Reserve stock |
| POST | `/api/v1/inventory/reservations/{reservationId}/release` | Release reservation |
| POST | `/api/v1/inventory/deductions` | Request stock deduction |
| POST | `/api/v1/adjustments` | Submit adjustment request |
| POST | `/api/v1/adjustments/{adjustmentId}/approve` | Approve adjustment |
| POST | `/api/v1/adjustments/{adjustmentId}/reject` | Reject adjustment |
| GET | `/api/v1/alerts` | Retrieve inventory alerts |
| POST | `/api/v1/alerts/{alertId}/acknowledge` | Acknowledge alert |
| POST | `/api/v1/alerts/{alertId}/resolve` | Resolve alert where permitted |
| GET | `/api/v1/inventory/transactions` | Retrieve stock transaction history |
| GET | `/api/v1/inventory/settings` | Retrieve branch inventory settings |
| PUT | `/api/v1/inventory/settings` | Update reorder settings |
| GET | `/api/v1/expiry-settings` | Retrieve expiry configuration |
| PUT | `/api/v1/expiry-settings` | Update expiry configuration |

Requirements:
- Stock updates must be atomic.
- A deduction must not make available stock negative.
- Expired batches must not be sold.
- Reservation and deduction operations must support idempotency.
- Branch scope must be derived from trusted identity and authorization rules.
- Batch identity and status must be validated using Catalog or trusted local reference data.
- Successful quantity changes must create stock transaction records.

The reservation API and exact state model require further design.

## 7. Stock Operations API

**Owner:** Stock Operations Service  
**Lead:** Sahan

### Receipts

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/receipts` | List receipts |
| GET | `/api/v1/receipts/{receiptId}` | Retrieve receipt |
| POST | `/api/v1/receipts` | Create receipt |
| POST | `/api/v1/receipts/{receiptId}/submit` | Submit for approval |
| POST | `/api/v1/receipts/{receiptId}/approve` | Approve receipt |
| POST | `/api/v1/receipts/{receiptId}/reject` | Reject receipt |

### Sales

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/sales` | List sales |
| GET | `/api/v1/sales/{saleId}` | Retrieve sale |
| POST | `/api/v1/sales` | Process sale |

Requirements:
- A receipt must not increase stock until it reaches the approved state.
- Publish `StockReceiptApproved` only after approval is committed.
- A sale must not be recorded as completed if stock deduction failed.
- Retain sale-time unit prices.
- Duplicate requests must not create duplicate sales or receipts.
- Validate branch and user scope.
- Coordinate with Inventory without directly modifying its database.

The exact sequence of local transaction, inventory request and event publication must be designed and tested. Do not assume a transaction can span two service databases.

## 8. Transfer API

**Owner:** Transfer Service

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/transfers` | List authorized transfers |
| GET | `/api/v1/transfers/{transferId}` | Retrieve transfer |
| POST | `/api/v1/transfers` | Create transfer request |
| POST | `/api/v1/transfers/{transferId}/approve` | Approve transfer |
| POST | `/api/v1/transfers/{transferId}/reject` | Reject transfer |
| POST | `/api/v1/transfers/{transferId}/dispatch` | Record dispatch |
| POST | `/api/v1/transfers/{transferId}/receive` | Confirm destination receipt |
| POST | `/api/v1/transfers/{transferId}/cancel` | Cancel where allowed |

Requirements:
- Source and destination branches must differ.
- Source manager must be authorized for the source branch.
- Destination receiver must be authorized for the destination branch.
- Requested quantity must be validated against available stock.
- Inventory handles reservations and quantity changes.
- Transfer records requested, approved, dispatched and received quantities.
- Partial dispatch, partial receipt and discrepancies must be handled explicitly.
- Repeated dispatch or receipt requests must not apply stock changes twice.

## 9. Notification API

**Owner:** Notification Service

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/v1/notifications` | List current user's notifications |
| GET | `/api/v1/notifications/{notificationId}` | Retrieve notification |
| PATCH | `/api/v1/notifications/{notificationId}/read` | Mark notification as read |
| POST | `/api/v1/notifications/read-all` | Mark current user's notifications as read |

A user may view and update only their own notifications unless a specific administrative policy permits otherwise.

## 10. Audit API and event intake

**Owner:** Audit Service

Proposed interface:

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/internal/api/v1/audit-events` | Submit audit event |
| GET | `/api/v1/audit-logs` | Search audit history |
| GET | `/api/v1/audit-logs/{auditId}` | Retrieve audit record |

Audit event fields should include:
- `eventId`
- `occurredAt`
- `actorUserId`
- `actionType`
- `entityName`
- `entityId`
- `description`
- `branchId` where relevant
- `correlationId`
- `sourceService`

The intake endpoint must be protected for service-to-service use. Audit retrieval must be restricted. Never include passwords, raw JWTs, refresh tokens or secrets.

Audit ingestion must be resilient. Decide whether the initial approach uses synchronous HTTP or asynchronous messaging. For important operations, consider a durable local outbox so an Audit outage does not silently lose records.

## 11. Event catalogue

Proposed event names:

| Event | Producer | Consumer(s) | Purpose |
|---|---|---|---|
| `StockReceiptApproved` | Stock Operations | Inventory | Apply accepted receipt quantities |
| `SaleProcessed` | Stock Operations | Inventory / Audit | Coordinate sale stock movement and audit |
| `TransferApproved` | Transfer | Inventory | Reserve source stock |
| `TransferDispatched` | Transfer | Inventory | Apply source stock movement |
| `TransferReceived` | Transfer | Inventory | Apply accepted destination quantities |
| `LowStockDetected` | Inventory | Notification / Audit | Notify relevant users |
| `NearExpiryDetected` | Inventory | Notification / Audit | Notify relevant users |
| `ExpiredStockDetected` | Inventory | Notification / Audit | Notify relevant users |
| Audit submission | Relevant services | Audit | Record important actions |

Each event must have one producer responsible for its meaning. Consumers must not infer facts that the producer has not explicitly provided.

## 12. Event envelope

Use a consistent envelope. Example:

```json
{
  "eventId": "unique-event-id",
  "eventType": "StockReceiptApproved",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T10:30:00Z",
  "producer": "stock-operations-service",
  "correlationId": "correlation-id",
  "causationId": "causation-id",
  "payload": {}
}
```

Use UTC ISO 8601 timestamps. `eventId` must be unique. Consumers should persist processed event IDs or use another durable idempotency mechanism.

## 13. Illustrative event payloads

The following examples are drafts, not finalized schemas.

### 13.1 StockReceiptApproved

```json
{
  "eventId": "event-1001",
  "eventType": "StockReceiptApproved",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T10:30:00Z",
  "producer": "stock-operations-service",
  "correlationId": "correlation-1001",
  "payload": {
    "receiptId": "receipt-501",
    "branchId": "B001",
    "approvedBy": "user-101",
    "items": [
      {
        "batchId": "batch-301",
        "acceptedQuantity": 50
      }
    ]
  }
}
```

Inventory validates and applies the stock increase once, then records the stock transaction.

### 13.2 SaleProcessed

```json
{
  "eventId": "event-1002",
  "eventType": "SaleProcessed",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T10:35:00Z",
  "producer": "stock-operations-service",
  "correlationId": "correlation-1002",
  "payload": {
    "saleId": "sale-801",
    "branchId": "B001",
    "soldBy": "user-102",
    "items": [
      {
        "batchId": "batch-301",
        "quantity": 2
      }
    ]
  }
}
```

**Important:** The team must decide whether Inventory deduction occurs synchronously before the sale is committed, or through a reservation/confirmation workflow. Publishing `SaleProcessed` after a sale is committed cannot alone guarantee that stock was deducted. The final workflow must prevent a completed sale with no corresponding stock deduction.

### 13.3 TransferDispatched

```json
{
  "eventId": "event-1003",
  "eventType": "TransferDispatched",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T11:00:00Z",
  "producer": "transfer-service",
  "correlationId": "correlation-1003",
  "payload": {
    "transferId": "transfer-701",
    "sourceBranchId": "B001",
    "destinationBranchId": "B002",
    "dispatchedBy": "user-103",
    "items": [
      {
        "batchId": "batch-301",
        "dispatchedQuantity": 10
      }
    ]
  }
}
```

Inventory processes the source branch movement idempotently. Destination stock is added only after receipt confirmation.

### 13.4 TransferReceived

```json
{
  "eventId": "event-1004",
  "eventType": "TransferReceived",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T13:00:00Z",
  "producer": "transfer-service",
  "correlationId": "correlation-1003",
  "payload": {
    "transferId": "transfer-701",
    "sourceBranchId": "B001",
    "destinationBranchId": "B002",
    "receivedBy": "user-104",
    "items": [
      {
        "batchId": "batch-301",
        "dispatchedQuantity": 10,
        "receivedQuantity": 9,
        "discrepancyReason": "One unit damaged during transit"
      }
    ]
  }
}
```

Inventory adds only the accepted quantity to destination stock according to the finalized discrepancy policy.

### 13.5 LowStockDetected

```json
{
  "eventId": "event-1005",
  "eventType": "LowStockDetected",
  "eventVersion": 1,
  "occurredAt": "2026-11-24T14:00:00Z",
  "producer": "inventory-service",
  "correlationId": "correlation-1005",
  "payload": {
    "alertId": "alert-901",
    "branchId": "B001",
    "medicineId": "medicine-201",
    "currentQuantity": 4,
    "reorderLevel": 10
  }
}
```

Notification determines appropriate recipients using trusted user/branch information and creates user-specific notifications.

## 14. Authentication and authorization

### 14.1 Authentication flow

1. Frontend submits credentials to the Gateway.
2. Gateway routes the request to Identity & Access.
3. Identity validates credentials and issues a signed JWT.
4. Frontend sends the access token on subsequent requests.
5. Gateway and receiving services validate the token according to the agreed trust configuration.

### 14.2 Initial token claims

The token may include:
- `sub`: stable user identifier.
- `role`: primary role.
- `branch_id`: assigned branch where applicable.
- `iss`: trusted issuer.
- `aud`: intended audience.
- `exp`: expiration.
- `iat`: issued-at time.
- `jti`: token identifier, if required.

Do not place passwords or sensitive personal information in JWT claims.

### 14.3 Authorization rules

- Identity & Access is authoritative for accounts and roles.
- The Gateway may apply coarse-grained policies.
- Each service enforces operation-level authorization.
- Branch scope comes from trusted claims or trusted service-side validation.
- A frontend-supplied branch ID is not proof of authorization.
- Service-to-service calls use authenticated service identity or another trusted mechanism.

## 15. Standard API response format

### Success

```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": {
    "id": "example-id"
  },
  "correlationId": "correlation-id"
}
```

### Error

```json
{
  "success": false,
  "message": "The requested operation could not be completed",
  "errorCode": "INSUFFICIENT_STOCK",
  "details": [],
  "correlationId": "correlation-id"
}
```

Do not expose stack traces, SQL errors, secrets or internal implementation details to clients.

## 16. HTTP status conventions

| Status | Usage |
|---|---|
| 200 OK | Successful retrieval or operation |
| 201 Created | Resource created |
| 202 Accepted | Asynchronous operation accepted |
| 204 No Content | Successful operation with no response body |
| 400 Bad Request | Invalid request or validation |
| 401 Unauthorized | Missing or invalid authentication |
| 403 Forbidden | Authenticated user lacks permission |
| 404 Not Found | Resource not found |
| 409 Conflict | State, duplicate or concurrency conflict |
| 422 Unprocessable Content | Semantically invalid business request, if adopted consistently |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Unexpected server error |
| 503 Service Unavailable | Required service temporarily unavailable |

Choose a consistent convention for 400 versus 422.

## 17. Reliability, idempotency and consistency

### Idempotency
Operations that may be retried must not create duplicate effects. Examples include sale creation, applying a receipt event, transfer dispatch, transfer receipt and stock deduction.

Use idempotency keys for suitable HTTP commands and unique `eventId` values for event consumers.

### Concurrency
Inventory changes must be protected against concurrent updates. Evaluate SQL transactions, optimistic concurrency tokens or appropriate locking.

### Retries
- Use bounded retries for transient failures.
- Use exponential backoff where appropriate.
- Do not retry permanent validation or authorization errors.
- Ensure non-idempotent operations have an idempotency strategy.

### Event delivery
Assume messages may be delivered more than once. Consumers must be idempotent. Failed messages must be observable and recoverable. A dead-letter queue may be used after the retry policy is exhausted.

### Distributed workflows
Do not assume one SQL transaction can cover multiple service databases. Use explicit workflow states, reservations, compensating actions or a Saga-style approach where necessary.

## 18. Audit integration

Relevant services submit audit information for important actions, including:
- User account creation or status changes.
- Medicine creation or modification.
- Receipt approval.
- Sale completion.
- Stock adjustment approval.
- Transfer approval, dispatch and receipt.
- Inventory setting changes.

Audit submissions should include actor, action, target entity, timestamp, source service and correlation ID.

The team must decide whether initial audit integration uses synchronous HTTP or asynchronous events. For important operations, a durable outbox is preferable to silently losing audit records when Audit is unavailable.

## 19. Cross-service workflow requirements

### 19.1 Receipt
- Operations owns receipt status and lines.
- Inventory owns stock balance changes.
- Operations publishes approval only after the approval is committed.
- Inventory processes the event once.
- Notification and Audit receive relevant information.

### 19.2 Sale
- Operations owns sale records and sale lines.
- Inventory owns stock deductions.
- The workflow must coordinate stock validation/deduction and sale completion.
- A completed sale must not exist without successful stock deduction.
- Use reservation/confirmation or another recoverable workflow.

### 19.3 Transfer
- Transfer owns request, approval, dispatch and receipt status.
- Inventory owns source reservation, source deduction and destination stock addition.
- Source and destination branches must differ.
- Only accepted destination quantities are added.
- Discrepancies must be recorded and auditable.

### 19.4 Alert and notification
- Inventory owns alert conditions and alert lifecycle.
- Notification owns user-specific notification records.
- One alert may create multiple notifications.
- Reading a notification does not resolve the alert.
- Alert resolution is controlled by Inventory business rules.

## 20. Implementation and testing checklist

### API contracts
- [ ] Every endpoint has an owner.
- [ ] Request and response DTOs are agreed.
- [ ] Authorization requirements are documented.
- [ ] Validation and error responses are consistent.
- [ ] API versioning is applied.

### Events
- [ ] Each event has one producer.
- [ ] Event envelope and payload are versioned.
- [ ] Event IDs are unique.
- [ ] Consumers are idempotent.
- [ ] Retry and failure handling are defined.
- [ ] Correlation IDs are propagated.

### Security
- [ ] JWT issuer, audience, signing and expiry are configured.
- [ ] Services validate trusted tokens.
- [ ] Branch scope is enforced server-side.
- [ ] Service-to-service calls are authenticated.
- [ ] Secrets are not committed to Git.

### Business workflows
- [ ] Receipt approval updates inventory exactly once.
- [ ] Sale completion and stock deduction remain consistent.
- [ ] Transfer reservation prevents overselling.
- [ ] Partial transfer receipt is handled.
- [ ] Expired batches cannot be sold.
- [ ] Alert and notification lifecycles remain separate.
- [ ] Important actions are auditable.

### Integration
- [ ] Each service can run independently.
- [ ] Each service has unit tests.
- [ ] Cross-service API tests exist.
- [ ] Event tests cover duplicate delivery.
- [ ] Failure and retry scenarios are tested.
- [ ] End-to-end workflows are tested.

## 21. Open decisions before implementation

1. API Gateway technology.
2. Message broker selection, including whether to use RabbitMQ.
3. Synchronous versus asynchronous audit submission for the first implementation.
4. Exact stock reservation and sale coordination workflow.
5. Exact transfer state machine, including partial dispatch and receipt.
6. Database-per-service deployment topology.
7. Event schemas and versioning rules.
8. Service-to-service authentication mechanism.
9. Exact role/permission matrix.
10. Whether services maintain local read models for branch and batch reference data.

## 22. Conclusion

This specification provides the initial communication contract for the seven Gagana Pharmacy services. It establishes service ownership, proposed REST endpoints, event responsibilities, security expectations and reliability requirements. Service leads should review their sections and agree on exact DTOs, status transitions and event payloads before parallel implementation begins.
