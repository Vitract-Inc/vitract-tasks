#Provider Kit Validation, Audit Logging & Provider Kits Page (Phase 1)

## 1. Executive Summary

Phase 1 introduces strict kit integrity controls and provider visibility features across the Virtual Providers system.

This phase ensures:

1. **Only valid kits** from the `valid_kits` table can be attached or updated on orders.
2. **All kit updates** are audit-logged for traceability and administrative resolution.
3. Providers gain a **scoped Kits page** to view the kits they are allowed to use, filtered by assigned kit types.

This is a data integrity and operational trust milestone. It reduces invalid registrations, prevents unauthorized kit usage, and creates accountability for post-registration edits.

---

## 2. Problem Statement

### Current Risks

* Providers can submit or update kit IDs without strict validation against a central source of truth.
* Updates after registration can create inconsistencies across downstream systems.
* There is no permanent audit trail for kit changes.
* Providers lack visibility into which kits they are allowed to use.

### Business Impact

* Invalid kits may enter the system.
* Registration conflicts occur.
* Admin teams lack traceability.
* Operational disputes are harder to resolve.

---

## 3. Objectives

### Primary Objectives

1. Enforce **centralized kit validation** using `valid_kits`.
2. Create an **append-only audit trail** for kit updates.
3. Provide providers with a **scoped Kits page** filtered by allowed kit types.

### Non-Goals (Phase 1)

* No automated conflict resolution.
* No admin alert workflow (logging only for now).
* No retroactive data cleanup.
* No redesign of registration flow.

---

## 4. Feature 1 — Kit Validation Enforcement

### 4.1 Source of Truth

All kits must exist in the `valid_kits` table and meet eligibility criteria.

`valid_kits` is the single authoritative dataset.

---

### 4.2 Scope of Validation

Validation must be enforced in **all provider endpoints** where kits are:

* Added to orders (including multi-kit submission)
* Updated on orders
* Edited through provider dashboard
* Submitted during shipment flows

Validation applies to **every kit in the request**, not just the first.

---

### 4.3 Validation Rules

For each kit:

1. Must exist in `valid_kits`
2. Must have an allowed status (e.g. `Issued`)
3. Must not be disallowed status (e.g. `Registered`, `Expired`, `Revoked`)
4. Must match the provider’s allowed `kit_type`

If any kit fails:

* Entire request fails
* Return clear error identifying invalid kit(s)

---

### 4.4 Provider-Level Kit Type Enforcement

Providers are assigned allowed `kit_type` values during creation.

Validation must also enforce:

```
kit_type IN provider.allowed_kit_types
```

This mirrors existing order visibility filtering logic.

---

### 4.5 Acceptance Criteria

* Submitting an invalid kit returns 400 error.
* Submitting a kit not allowed for that provider returns 403 error.
* Multi-kit request fails if any kit is invalid.
* Validation works consistently across all endpoints.

---

## 5. Feature 2 — Order Kits Update Logging (Audit Trail)

### 5.1 Purpose

Create a permanent record of kit updates to ensure traceability and enable administrative resolution.

---

### 5.2 When to Log

Log entries are created:

* Only when kits are updated
* Not when kits are initially added

---

### 5.3 Special Case: Post-Registration Updates

If kits are updated after registration:

* Update is allowed only in `order_kits`
* No downstream propagation
* A log entry must be created
* Admins will resolve manually (Phase 2)

---

### 5.4 Logging Table: `order_kits_update_logs`

Append-only table.

Minimum required fields:

* `id` (UUID)
* `order_id`
* `provider_id`
* `previous_kit_ids` (array or JSON snapshot)
* `new_kit_ids` (array or JSON snapshot)
* `registration_status`
* `source` (provider_api | dashboard)
* `correlation_id`
* `updated_at`

Optional but recommended:

* `reason`
* `actor_type`

---

### 5.5 System Behavior

* Log entry must be created in the same transaction as update.
* Logs are immutable.
* No updates or deletes allowed.

---

### 5.6 Acceptance Criteria

* Updating kits creates exactly one log entry.
* Adding kits creates no log entry.
* Logs persist even if update affects registered kits.
* Logs include actor identity.

---

## 6. Feature 3 — Provider Kits Page

### 6.1 Purpose

Allow providers to view the kits they are authorized to use.

This page will be called:

```
Kits
```

(Not “Valid Kits”)

---

### 6.2 Data Source

Table: `valid_kits`

Filtered by:

```
valid_kits.kit_type IN provider.allowed_kit_types
AND valid_kits.status = 'Issued'
```

---

### 6.3 Backend Endpoint

```
GET /provider/kits
```

Query parameters:

* `page`
* `pageSize`
* `search` (kit ID)
* `kitType` (optional filter within allowed types)

Server must enforce allowed types regardless of request parameters.

---

### 6.4 Frontend Requirements

Page: `/provider/kits`

Features:

* Paginated table
* Search by kit ID (debounced)
* Filter by kit type (restricted to provider’s allowed types)
* Status column
* Issued date (if available)

---

### 6.5 Performance Requirements

Minimum DB requirements:

* Index on `kit_id`
* Index on `(kit_type, status)`
* Pagination must not load entire dataset

---

### 6.6 Acceptance Criteria

* Provider only sees kits matching assigned kit types.
* Provider cannot manipulate query to view other kit types.
* Search works within allowed dataset.
* Response time < 300ms for normal page sizes.

---

## 7. Security & Authorization

### Provider Constraints

* Providers may only update kits on their own orders.
* Providers may only see kits for assigned `kit_type`.
* All validation must be enforced server-side.

---

## 8. Rollout Plan

1. Create DB migration:

   * `order_kits_update_logs`
   * Add required indexes
2. Deploy validation enforcement
3. Deploy logging
4. Deploy provider Kits page
5. Monitor:

   * Validation failure rate
   * Update log volume
   * Provider usage metrics

---

## 9. Success Metrics

* 100% of provider kit submissions validated.
* 0 invalid kits entering registration pipeline.
* 100% update actions logged.
* No cross-provider kit visibility incidents.

---

## 10. Future Phases (Out of Scope)

* Admin dashboard for resolving update conflicts
* Automatic alerting on post-registration edits
* Automated downstream reconciliation
* Real-time provider inventory tracking
