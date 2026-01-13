# PRD: Automated Practitioner Kit Registration & Health Information Dispatch

## Executive Summary

This project introduces **automatic practitioner kit registration** and **delayed health information dispatch** to eliminate manual operational steps after an order is shipped.

Historically, practitioners manually registered kits and entered the **sample collection date** during registration. With the introduction of **auto-registration**, that manual step is removed, which also removes the original point where sample collection date was captured.

To preserve required data integrity without re-introducing manual registration, the **sample collection date will now be collected from the client as part of the Health Information Form flow**, before they proceed to answer health questions.

This design:

* Keeps registration fully automated
* Ensures required sample metadata is always captured
* Preserves a clean and scalable workflow across backend and frontend

---

## 1. Background & Problem Statement

Currently, practitioner orders on the Vitract platform require **manual kit registration and manual health-information sending**. This creates delays and unnecessary operational overhead.

We want to **fully automate**:

1. **Kit registration (step 1)** immediately after an order is shipped
2. **Health information form dispatch** automatically **one day after registration**, without manual intervention

With auto-registration in place, practitioners will no longer manually register kits. As a result, the system must introduce a new point to collect **sample collection date**, which was previously captured during manual registration.

This must work reliably even when:

* Orders contain **multiple kits**
* Jobs are retried
* Systems restart
* Both `vitract-orders` and `vitract-kit-api-v1` update shipment status independently

---

## 2. Goals

### Primary Goals

* Automatically register practitioner order kits (step 1) when orders are shipped
* Automatically send health information forms **after a delay**
* Preserve capture of **sample collection date** despite removal of manual registration
* Ensure **idempotency** and **fault tolerance**
* Maintain a durable audit trail for dispatch attempts

### Non-Goals

* Manual UI-driven kit registration
* Client-initiated kit flows
* Health-information form completion logic beyond required metadata capture

---

## 3. Scope of Work (Repositories & UI)

### In Scope

#### Backend

* `vitract-orders`
* `vitract-kit-api-v1`
* Queues, cron jobs, dispatch logs

#### Frontend

* Health Information Form entry flow
* Sample collection date capture

### Out of Scope

* Redesign of health questionnaire content
* Email template design (assumed existing)

---

## 4. High-Level Architecture

```
Order Shipped
     │
     ▼
AUTO_REGISTER_PRACTIONER_ORDER_KITS Queue
     │
     ├─ Register kit(s) (step 1)
     └─ Create health_information_dispatch_logs (PENDING)
                │
                ▼
Hourly Cron
     │
     ▼
HEALTH_INFORMATION_DISPATCH Queue
     │
     └─ Send health info email → Update status
```

---

## 5. Queue Definitions

### Queue 1: `AUTO_REGISTER_PRACTIONER_ORDER_KITS`

**Triggered by**

* Order shipment update in:

  * `vitract-orders`
  * `vitract-kit-api-v1`

**Purpose**

* Register kits (step 1)
* Create health information dispatch log entries

---

### Queue 2: `HEALTH_INFORMATION_DISPATCH`

**Triggered by**

* Hourly cron job

**Purpose**

* Send health information forms for eligible kits
* Update dispatch status and retry metadata

---

## 6. Database Schema

### Table: `health_information_dispatch_logs`

| Column         | Type     | Notes                               |        |      |        |            |
| -------------- | -------- | ----------------------------------- | ------ | ---- | ------ | ---------- |
| id             | uuid     | Primary key                         |        |      |        |            |
| orderId        | uuid     | Indexed                             |        |      |        |            |
| kitId          | varchar  | One row per kit                     |        |      |        |            |
| practitionerId | uuid     | Order owner                         |        |      |        |            |
| recipientEmail | varchar  | Email to send form to               |        |      |        |            |
| registeredAt   | datetime | Set when kit registration completes |        |      |        |            |
| status         | varchar  | `PENDING                            | QUEUED | SENT | FAILED | CANCELLED` |
| sentAt         | datetime | Nullable                            |        |      |        |            |
| attempts       | int      | Default 0                           |        |      |        |            |
| lastError      | text     | Nullable                            |        |      |        |            |
| createdAt      | datetime |                                     |        |      |        |            |
| updatedAt      | datetime |                                     |        |      |        |            |

#### Constraints

* **Unique**: `(orderId, kitId)`
* **Index**: `(status, registeredAt)`

---

## 7. Entity Relationships (Existing)

* `Order` → `OrderKit` (one-to-many)
* Each `OrderKit` produces **one dispatch log row**
* Orders may contain **multiple kits**

---

## 8. Trigger Conditions

### When to enqueue `AUTO_REGISTER_PRACTIONER_ORDER_KITS`

Triggered when:

* Order shipment fields are updated:

  * `shippingDate`
  * `trackingNumber`
  * `trackingUrl`
  * `status = shipped`
* Order belongs to a **practitioner**
* Triggered in **both repositories**

Payload:

```ts
{
  orderId: string
}
```

---

## 9. Processor Logic

### Processor: AUTO_REGISTER_PRACTIONER_ORDER_KITS

**Steps**

1. Fetch `Order` + `OrderKits`
2. For each `OrderKit`:

   * Register kit (step 1)
   * Create `health_information_dispatch_logs` row:

     * `registeredAt = now()`
     * `status = PENDING`
3. Update order registration fields if applicable

**Idempotency**

* If `(orderId, kitId)` exists → skip
* Safe to retry

---

### Processor: HEALTH_INFORMATION_DISPATCH

**Cron Frequency**

* Every hour

**Selection Criteria**

```sql
registeredAt + INTERVAL '1 day' <= now()
AND status = 'PENDING'
```

**Steps**

1. Atomically update status → `QUEUED`
2. Send health information email
3. On success:

   * `status = SENT`
   * `sentAt = now()`
4. On failure:

   * `status = FAILED`
   * `attempts += 1`
   * `lastError = error message`

**Concurrency Safety**

* Update includes `WHERE status = 'PENDING'`

---

## 10. Cron Responsibilities

* Cron does **not** send emails
* Only enqueues `HEALTH_INFORMATION_DISPATCH` jobs
* Supports batching and throttling

---

## 11. Error Handling & Retries

* Failures recorded per kit
* Attempts incremented
* Failed rows may be retried or reviewed manually

---

## 12. Observability & Debugging

* Full audit trail per kit
* Clear state transitions:
  `PENDING → QUEUED → SENT / FAILED`
* Order → Kit → Dispatch traceability

---

## 13. Frontend Requirement: Health Information Form Update (NEW)

### Context

With auto-registration, practitioners no longer manually register kits.
The **sample collection date**, previously captured during registration, must now be collected during the **Health Information Form flow**.

---

### Required Behavior

#### Health Information Form – Entry Page

Before accessing any health question categories:

1. **Sample Collection Date**

   * Required field
   * Collected once per kit
   * Persisted immediately

2. **Health Question Categories**

   * Displayed only after sample collection date is provided
   * Existing category-based flow remains unchanged

---

### User Flow

```
Health Information Form (Landing Page)
│
├─ Sample Collection Date (required)
│
└─ Health Question Categories
      ├─ Category A
      ├─ Category B
      └─ Category C
```

---

### Validation Rules

* Sample collection date:

  * Required
  * Valid date
  * Must be saved before entering questions

---

## 14. Implementation Checklist

### vitract-orders

* [ ] Add queues module
* [ ] Add `AUTO_REGISTER_PRACTIONER_ORDER_KITS` processor
* [ ] Add `HEALTH_INFORMATION_DISPATCH` processor
* [ ] Add migration + entity for `health_information_dispatch_logs`
* [ ] Enqueue auto-register job on shipment update

### vitract-kit-api-v1

* [ ] Enqueue `AUTO_REGISTER_PRACTIONER_ORDER_KITS` on shipment update

### Frontend

* [ ] Add sample collection date field to health information form entry page
* [ ] Enforce required validation
* [ ] Persist date per kit before question flow

---

## 15. Acceptance Criteria

* Practitioner kits auto-register on shipment
* Health information emails send **exactly once per kit**
* Sample collection date is always captured despite removal of manual registration
* Delayed dispatch survives restarts
* No duplicate sends
* Logs reflect true state at all times
