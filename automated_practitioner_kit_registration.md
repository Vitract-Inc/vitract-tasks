# PRD: Automated Practitioner Kit Registration & Health Information Dispatch

## 1. Background & Problem Statement

Currently, practitioner orders on the Vitract platform require **manual kit registration and manual health-information sending**. This creates delays, inconsistency, and unnecessary operational overhead.

We want to **fully automate**:

1. **Kit registration (step 1)** immediately after an order is shipped.
2. **Health information form dispatch** automatically **one day after registration**, without manual intervention.

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
* Ensure **idempotency** and **fault tolerance**
* Maintain a durable audit trail for dispatch attempts

### Non-Goals

* Manual UI-driven registration
* Client-initiated kit flows
* Health-information form completion logic (sending only)

---

## 3. Scope of Work (Repositories)

### In Scope

* `vitract-orders`
* `vitract-kit-api-v1`

### Out of Scope

* Frontend health-info UI
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

| Column         | Type     | Notes                                        |        |      |        |            |
| -------------- | -------- | -------------------------------------------- | ------ | ---- | ------ | ---------- |
| id             | uuid     | Primary key                                  |        |      |        |            |
| orderId        | uuid     | Indexed                                      |        |      |        |            |
| kitId          | varchar  | One row per kit                              |        |      |        |            |
| practitionerId | uuid     | Order owner                                  |        |      |        |            |
| recipientEmail | varchar  | Email to send form to                        |        |      |        |            |
| registeredAt   | datetime | Indexed; set when kit registration completes |        |      |        |            |
| status         | varchar  | `PENDING                                     | QUEUED | SENT | FAILED | CANCELLED` |
| sentAt         | datetime | Nullable                                     |        |      |        |            |
| attempts       | int      | Default 0                                    |        |      |        |            |
| lastError      | text     | Nullable                                     |        |      |        |            |
| createdAt      | datetime |                                              |        |      |        |            |
| updatedAt      | datetime |                                              |        |      |        |            |

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
* Triggered in **both repositories** (`vitract-orders`, `vitract-kit-api-v1`)

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

   * Register kit (step 1) using existing kit logic
   * Create `health_information_dispatch_logs` row:

     * `registeredAt = now()`
     * `status = PENDING`
3. Update order registration fields if applicable

**Idempotency**

* If `(orderId, kitId)` exists → skip
* Safe to retry without duplication

---

### Processor: HEALTH_INFORMATION_DISPATCH

**Cron Frequency**

* Every hour

**Selection Criteria**

```sql
registeredAt + INTERVAL '1 day' <= now()
AND status = 'PENDING'
```

**Steps per record**

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

* Status update must include `WHERE status = 'PENDING'`
* If update affects 0 rows → already claimed

---

## 10. Cron Responsibilities

The cron job:

* Does **not** send emails directly
* Only enqueues `HEALTH_INFORMATION_DISPATCH` jobs
* Can be throttled or batched later

---

## 11. Error Handling & Retries

* Failures are recorded per kit
* Attempts are incremented
* Re-dispatch logic can:

  * Retry FAILED rows later
  * Or leave FAILED for manual review

---

## 12. Observability & Debugging

* Every dispatch attempt is logged
* Clear state transitions: `PENDING → QUEUED → SENT / FAILED`
* Order → Kit → Dispatch traceability

---

## 13. Implementation Checklist

### vitract-orders

* [ ] Add queues module (matching kit-api-v1 structure)
* [ ] Add `AUTO_REGISTER_PRACTIONER_ORDER_KITS` processor
* [ ] Add `HEALTH_INFORMATION_DISPATCH` processor
* [ ] Add migration + entity for `health_information_dispatch_logs`
* [ ] Enqueue auto-register job on shipment update

### vitract-kit-api-v1

* [ ] Enqueue `AUTO_REGISTER_PRACTIONER_ORDER_KITS` on shipment update

---

## 14. Acceptance Criteria

* Practitioner orders auto-register kits on shipment
* Health information emails send **exactly once per kit**
* Delayed dispatch works across restarts
* No duplicate sends
* Logs reflect true state at all times
