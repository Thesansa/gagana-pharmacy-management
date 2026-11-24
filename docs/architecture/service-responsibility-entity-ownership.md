# Gagana Pharmacy Management System
# Service Responsibility & Entity Ownership Matrix

**Status:** Proposed ownership baseline  
**Version:** 1.0  
**Last updated:** 24 November 2026  
**Related document:** `microservice-architecture-v1.md`

---

## 1. Purpose

This document maps the current 22 core entities to the seven proposed microservices. It identifies the authoritative owner of each entity, local relationships that can use SQL foreign keys, and references that cross service boundaries.

The original EERD remains the logical business model. This document describes how its entities are allocated to service-owned databases.

## 2. Ownership rules

1. Every core entity has exactly one owning service.
2. Only the owning service may create, update or delete its authoritative records.
3. A service must not directly access another service's database tables.
4. Relationships inside one service database may use normal SQL Server foreign keys.
5. Cross-service relationships use identifiers and APIs or trusted events.
6. A local copy/read model is not authoritative unless ownership is explicitly transferred.
7. Cross-service consistency is handled through validation, events and workflow design.

## 3. Ownership summary

| Service | Core entities | Count |
|---|---|---:|
| Identity & Access | `BRANCH`, `DEPARTMENT`, `EMPLOYEE`, `ROLE`, `USER_ACCOUNT` | 5 |
| Audit | `AUDIT_LOG` | 1 |
| Inventory Management | `BRANCH_INVENTORY`, `BRANCH_MEDICINE_SETTING`, `EXPIRY_ALERT_SETTING`, `ALERT`, `STOCK_ADJUSTMENT`, `STOCK_TRANSACTION` | 6 |
| Medicine Catalog | `MEDICINE`, `MEDICINE_BATCH`, `BATCH_PRICE_HISTORY` | 3 |
| Stock Operations | `STOCK_RECEIPT`, `STOCK_RECEIPT_ITEM`, `SALE`, `SALE_ITEM` | 4 |
| Transfer | `STOCK_TRANSFER`, `STOCK_TRANSFER_ITEM` | 2 |
| Notification | `NOTIFICATION` | 1 |
| **Total** | | **22** |

The three alert specialization entities (`LOW_STOCK_ALERT`, `NEAR_EXPIRY_ALERT`, `EXPIRED_STOCK_ALERT`) are subtypes of `ALERT` in the conceptual EERD. They are included within Inventory Management's alert model and are not counted as additional core entities above. If implemented as separate relational tables, they remain in the Inventory database.

## 4. Detailed entity ownership

### 4.1 Identity & Access Service

**Database:** Identity DB  
**Lead:** Project technical lead

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `BRANCH` | `branch_id`, `branch_name`, `address`, `contact_number`, `status` | Branch master data |
| `DEPARTMENT` | `department_id`, `department_name`, `description` | Department master data |
| `EMPLOYEE` | `employee_id`, `branch_id`, `department_id`, employee details, `employment_status` | Employee records and assignment |
| `ROLE` | `role_id`, `role_name`, `description` | Primary application roles |
| `USER_ACCOUNT` | `user_id`, `employee_id`, `role_id`, `username`, `password_hash`, `account_status` | Accounts and credentials |

**Confirmed assumptions:**
- Each user account has one primary role. `USER_ACCOUNT.role_id` references `ROLE`.
- A separate `USER_ROLE` junction table is not required for the current one-role model.
- Each employee belongs to one branch. `EMPLOYEE.branch_id` references `BRANCH`.
- An Admin account may exist without an employee record; `USER_ACCOUNT.employee_id` may therefore be nullable.
- `USER_ACCOUNT.employee_id` should be unique if one employee can have at most one account.
- Branch and employee management belong to Identity & Access.

**Local relationships:**
- `EMPLOYEE.branch_id` → `BRANCH.branch_id`
- `EMPLOYEE.department_id` → `DEPARTMENT.department_id`
- `USER_ACCOUNT.employee_id` → `EMPLOYEE.employee_id`
- `USER_ACCOUNT.role_id` → `ROLE.role_id`

### 4.2 Audit Service

**Database:** Audit DB  
**Lead:** Project technical lead

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `AUDIT_LOG` | `audit_id`, `user_id`, `action_type`, `entity_name`, `entity_id`, `action_description`, `action_timestamp` | Audit history |

`user_id` is an external reference to Identity & Access, not a cross-database FK.

`entity_name` and `entity_id` identify the affected record. Since the record may belong to different services, this is a polymorphic reference rather than a conventional FK to multiple tables.

Audit records should not contain passwords, raw access tokens or secrets.

### 4.3 Inventory Management Service

**Database:** Inventory DB  
**Lead:** Project technical lead

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `BRANCH_INVENTORY` | `inventory_id`, `branch_id`, `batch_id`, `available_quantity`, `reserved_quantity`, `damaged_quantity`, `last_updated` | Authoritative stock quantities |
| `BRANCH_MEDICINE_SETTING` | `setting_id`, `branch_id`, `medicine_id`, `reorder_level`, `setting_updated_at` | Branch reorder thresholds |
| `EXPIRY_ALERT_SETTING` | `expiry_alert_setting_id`, `branch_id`, `expiry_alert_days`, `updated_by`, `updated_at` | Expiry monitoring configuration |
| `ALERT` | `alert_id`, `branch_id`, `alert_type`, `alert_message`, `created_at`, `alert_status` | Inventory alert lifecycle |
| `STOCK_ADJUSTMENT` | `adjustment_id`, `branch_id`, `batch_id`, `requested_by`, `approved_by`, `adjustment_type`, `adjustment_quantity`, `adjustment_status` | Adjustment workflow |
| `STOCK_TRANSACTION` | `transaction_id`, `branch_id`, `batch_id`, `performed_by`, `transaction_type`, `quantity_changed`, `transaction_date`, `source_operation_type`, `source_operation_id` | Stock movement ledger |

**Alert subtypes:**
- `LOW_STOCK_ALERT`
- `NEAR_EXPIRY_ALERT`
- `EXPIRED_STOCK_ALERT`

These remain part of the Inventory Service's alert model. If represented as subtype tables, each subtype can use `alert_id` as both PK and FK to `ALERT`.

**External references:**
- `branch_id` refers to a branch owned by Identity & Access.
- `batch_id` refers to a batch owned by Catalog.
- `medicine_id` refers to a medicine owned by Catalog.
- `requested_by`, `approved_by`, `performed_by` and `updated_by` refer to users/employees owned by Identity & Access.

**Constraints to consider:**
- Unique `(branch_id, batch_id)` in `BRANCH_INVENTORY`.
- Unique `(branch_id, medicine_id)` in `BRANCH_MEDICINE_SETTING`.
- Non-negative quantity checks.
- A clear definition of whether `quantity_changed` is signed or direction is represented by `transaction_type`.
- Alert subtype consistency with `alert_type`.

`source_operation_type` and `source_operation_id` form a polymorphic reference to the originating workflow, not a conventional FK.

### 4.4 Medicine Catalog Service

**Database:** Catalog DB  
**Lead:** Nethmini

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `MEDICINE` | `medicine_id`, `medicine_name`, `generic_name`, `category`, `description` | Medicine master data |
| `MEDICINE_BATCH` | `batch_id`, `medicine_id`, `batch_number`, `manufacturing_date`, `expiry_date`, `selling_price` | Batch and expiry data |
| `BATCH_PRICE_HISTORY` | `price_history_id`, `batch_id`, `changed_by`, `old_price`, `new_price`, `change_reason`, `changed_at` | Historical price changes |

**Local relationships:**
- `MEDICINE_BATCH.medicine_id` → `MEDICINE.medicine_id`
- `BATCH_PRICE_HISTORY.batch_id` → `MEDICINE_BATCH.batch_id`

`changed_by` is an external user reference to Identity & Access.

A candidate key such as `(medicine_id, batch_number)` is valid only if uniqueness is a business rule and is enforced by the database.

### 4.5 Stock Operations Service

**Database:** Operations DB  
**Lead:** Sahan

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `STOCK_RECEIPT` | `receipt_id`, `branch_id`, `received_by`, `approved_by`, `receipt_date`, `reference_number`, `receipt_status` | Incoming receipt |
| `STOCK_RECEIPT_ITEM` | `receipt_item_id`, `receipt_id`, `batch_id`, `delivered_quantity`, `accepted_quantity`, `rejected_quantity`, `rejection_reason` | Receipt lines |
| `SALE` | `sale_id`, `branch_id`, `sold_by`, `sale_date`, `total_amount` | Sale header |
| `SALE_ITEM` | `sale_item_id`, `sale_id`, `batch_id`, `quantity`, `unit_price`, `batch_selection_method` | Sale lines |

**Local relationships:**
- `STOCK_RECEIPT_ITEM.receipt_id` → `STOCK_RECEIPT.receipt_id`
- `SALE_ITEM.sale_id` → `SALE.sale_id`

**External references:**
- `branch_id` refers to Identity & Access.
- `batch_id` refers to Catalog.
- `received_by`, `approved_by` and `sold_by` refer to Identity & Access.

`SALE_ITEM.unit_price` should preserve the price charged at the time of sale. It must not change when Catalog updates the current batch price.

If `SALE.total_amount` is stored, it must be kept consistent with its sale items. Alternatively, calculate it from the lines when queried.

### 4.6 Transfer Service

**Database:** Transfer DB  
**Lead:** Supporting pair under technical lead review

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `STOCK_TRANSFER` | `transfer_id`, `source_branch_id`, `destination_branch_id`, `requested_by`, `approved_by`, `received_by`, `transfer_date`, `dispatch_date`, `received_date`, `transfer_status` | Transfer lifecycle |
| `STOCK_TRANSFER_ITEM` | `transfer_item_id`, `transfer_id`, `batch_id`, `requested_quantity`, `dispatched_quantity`, `received_quantity`, `discrepancy_reason` | Transfer lines |

**Local relationship:**
- `STOCK_TRANSFER_ITEM.transfer_id` → `STOCK_TRANSFER.transfer_id`

**External references:**
- Source and destination branch IDs refer to Identity & Access.
- `batch_id` refers to Catalog.
- `requested_by`, `approved_by` and `received_by` refer to Identity & Access.

Rules:
- Source and destination branches must differ.
- Requested quantities must be positive.
- Dispatched and received quantities must follow the approved workflow.
- Inventory owns stock reservation and quantity changes.
- Transfer owns request, approval and confirmation records.

### 4.7 Notification Service

**Database:** Notification DB  
**Lead:** Supporting pair under technical lead review

| Entity | Illustrative attributes | Responsibility |
|---|---|---|
| `NOTIFICATION` | `notification_id`, `user_id`, `alert_id` (nullable), `notification_type`, `message`, `created_at`, `read_status` | User-specific notifications |

**External references:**
- `user_id` refers to a user account owned by Identity & Access.
- `alert_id`, when present, refers to an alert owned by Inventory Management.

One alert can produce multiple notification records. Read status belongs to each notification, not to the shared alert.

## 5. Cross-service relationship matrix

| Owning service | Field | Referenced service | Referenced entity | Validation approach |
|---|---|---|---|---|
| Audit | `AUDIT_LOG.user_id` | Identity | `USER_ACCOUNT` | Trusted identity context / API |
| Inventory | `BRANCH_INVENTORY.branch_id` | Identity | `BRANCH` | API or trusted branch data |
| Inventory | `BRANCH_INVENTORY.batch_id` | Catalog | `MEDICINE_BATCH` | Catalog API / trusted event |
| Inventory | `BRANCH_MEDICINE_SETTING.medicine_id` | Catalog | `MEDICINE` | Catalog API |
| Inventory | `EXPIRY_ALERT_SETTING.updated_by` | Identity | User/employee | Trusted identity context |
| Inventory | `STOCK_ADJUSTMENT.branch_id` | Identity | `BRANCH` | API or trusted branch data |
| Inventory | `STOCK_ADJUSTMENT.batch_id` | Catalog | `MEDICINE_BATCH` | Catalog API |
| Inventory | Adjustment actor IDs | Identity | User/employee | Trusted identity context |
| Inventory | `STOCK_TRANSACTION.branch_id` | Identity | `BRANCH` | Trusted branch data |
| Inventory | `STOCK_TRANSACTION.batch_id` | Catalog | `MEDICINE_BATCH` | Catalog API / trusted event |
| Operations | `STOCK_RECEIPT.branch_id` | Identity | `BRANCH` | API or trusted branch data |
| Operations | Receipt actor IDs | Identity | User/employee | Trusted identity context |
| Operations | `STOCK_RECEIPT_ITEM.batch_id` | Catalog | `MEDICINE_BATCH` | Catalog API |
| Operations | `SALE.branch_id` | Identity | `BRANCH` | Trusted branch data |
| Operations | `SALE.sold_by` | Identity | User/employee | Trusted identity context |
| Operations | `SALE_ITEM.batch_id` | Catalog | `MEDICINE_BATCH` | Catalog API |
| Transfer | Source/destination branch IDs | Identity | `BRANCH` | API or trusted branch data |
| Transfer | Transfer actor IDs | Identity | User/employee | Trusted identity context |
| Transfer | `STOCK_TRANSFER_ITEM.batch_id` | Catalog | `MEDICINE_BATCH` | Catalog API |
| Notification | `NOTIFICATION.user_id` | Identity | `USER_ACCOUNT` | Trusted identity context / API |
| Notification | `NOTIFICATION.alert_id` | Inventory | `ALERT` | Event payload / API |

Not every operation needs a synchronous validation call if the service already has trusted, versioned reference data from events.

## 6. Local foreign keys versus external references

### Local foreign keys

These relationships remain within one service database:

**Identity DB**
- `EMPLOYEE.branch_id` → `BRANCH.branch_id`
- `EMPLOYEE.department_id` → `DEPARTMENT.department_id`
- `USER_ACCOUNT.employee_id` → `EMPLOYEE.employee_id`
- `USER_ACCOUNT.role_id` → `ROLE.role_id`

**Catalog DB**
- `MEDICINE_BATCH.medicine_id` → `MEDICINE.medicine_id`
- `BATCH_PRICE_HISTORY.batch_id` → `MEDICINE_BATCH.batch_id`

**Operations DB**
- `STOCK_RECEIPT_ITEM.receipt_id` → `STOCK_RECEIPT.receipt_id`
- `SALE_ITEM.sale_id` → `SALE.sale_id`

**Transfer DB**
- `STOCK_TRANSFER_ITEM.transfer_id` → `STOCK_TRANSFER.transfer_id`

**Inventory DB**
- Alert subtype `alert_id` → `ALERT.alert_id`, if subtype tables are used.
- Other Inventory-local relationships follow the final relational schema.

### External references

External references are identifiers stored by a service for records owned elsewhere. They are not cross-database foreign keys. The owning service provides APIs or events for validation and reference data.

## 7. Important modelling considerations

### 7.1 Alert versus Notification

- `ALERT` belongs to Inventory and represents an inventory condition.
- `NOTIFICATION` belongs to Notification and represents a message delivered to a user.
- One alert can create multiple notifications.
- An administrative notification may have no `alert_id`.
- Viewing a notification does not automatically resolve an alert.

### 7.2 Audit log versus stock transaction

- `AUDIT_LOG` answers: who performed an action, and when?
- `STOCK_TRANSACTION` answers: what happened to a stock quantity?

They must remain separate.

### 7.3 Stock balance ownership

Only Inventory Management updates authoritative stock quantities. Operations and Transfer request inventory actions through defined APIs or events.

### 7.4 Polymorphic references

`AUDIT_LOG.entity_name` + `entity_id` and `STOCK_TRANSACTION.source_operation_type` + `source_operation_id` can point to records in different services. They are polymorphic references, not ordinary foreign keys. Their validity is maintained through application contracts and traceability.

### 7.5 Historical snapshots

Store values needed to preserve historical accuracy. For example, `SALE_ITEM.unit_price` is a sale-time snapshot and should not be reconstructed from the current Catalog price.

## 8. Ownership validation checklist

- [ ] Every entity has exactly one owning service.
- [ ] Each service has its own database ownership boundary.
- [ ] No service directly writes to another service's tables.
- [ ] Local foreign keys are identified.
- [ ] Cross-service references are identified.
- [ ] User and branch references use trusted identity information.
- [ ] Batch references are validated against Catalog.
- [ ] Inventory is the only authority for stock balances.
- [ ] Alert and Notification remain separate concepts.
- [ ] Audit and Stock Transaction remain separate concepts.
- [ ] Event consumers are idempotent.

## 9. Items to verify against the final EERD

This matrix follows the current 22-entity model and confirmed assumptions. Before generating implementation SQL, verify exact attributes, keys and cardinalities against the final approved EERD and relational schema.

In particular:
- Confirm final field names and data types.
- Confirm whether alert subtype tables are implemented separately.
- Confirm whether `USER_ACCOUNT.employee_id` is nullable and unique.
- Confirm candidate-key uniqueness rules for batch numbers and branch inventory combinations.
- Confirm the meaning and sign convention of stock transaction quantities.
- Confirm final approval and receipt workflow fields.
- Confirm whether `SALE.total_amount` is stored or derived.

## 10. Conclusion

This matrix allocates the current 22 core entities across seven services. It preserves local relational integrity within each service while replacing cross-service foreign keys with explicit identifiers, APIs and events. It is the basis for database-per-service design, API contracts, event contracts and parallel team implementation.
