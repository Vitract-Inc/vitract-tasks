# Automated Practitioner Kit Registration & Clinic Workflow Completion

---

## Executive Summary

This project completes the **Automated Practitioner Kit Registration** initiative by ensuring that automation works correctly alongside real-world clinic workflows, manual registrations, and client-driven registration paths.

Historically, practitioners manually registered kits after shipment. This ensured clear ownership and visibility but introduced operational delays and support overhead. Automation has since been introduced for kits shipped directly to patients, removing the need for manual registration in the majority of cases.

However, introducing automation surfaced several critical edge cases:

* Clinics often order kits in bulk and hand them to patients on-site
* Practitioners need clear guidance on when manual registration is still required
* Clients may register kits themselves, unintentionally blocking practitioner visibility

This project resolves those gaps by:

* Explicitly separating **dropship** and **on-site** workflows
* Clarifying manual registration intent in the practitioner platform
* Automatically preserving practitioner visibility when clients register practitioner-purchased kits
* Accurately recording **who registered each kit and how**

The result is a system that supports automation **without breaking clinic operations**, eliminates common visibility issues, and reduces ongoing support intervention.

---

## 1. Background & Problem Statement

Automated registration is now used for practitioner orders where kits are shipped directly to patients. Once shipped, these kits are registered automatically by the system, and downstream processes continue without practitioner involvement.

However, not all practitioner orders follow this pattern. In real-world clinic workflows:

* Kits may be shipped to clinics
* Kits may be handed to patients in person
* Registration may occur later and may be performed by either the practitioner or the client

Without explicit handling, these cases introduce:

* Premature registration
* Confusion around manual responsibilities
* Loss of practitioner visibility when clients register kits themselves

The system must correctly support **all registration pathways** while maintaining auditability, clarity, and reliability.

---

## 2. Goals

### Primary Goals

* Fully support both **dropship** and **on-site clinic** workflows
* Preserve automation where appropriate
* Ensure practitioners always retain visibility into kits they purchased
* Correctly attribute how and by whom each kit was registered
* Eliminate support cases caused by missing access or ambiguous workflows

### Non-Goals

* Changing laboratory workflows
* Redesigning health questionnaires
* Removing client self-registration
* Introducing billing or payment changes

---

## 3. Scope of Work (Backend & UI)

### In Scope

**Backend**

* Order registration handling
* Order-kit registration audit updates
* Practitioner visibility enforcement
* Registration source attribution

**Frontend**

* Practitioner platform navigation clarity
* Manual registration intent communication

### Out of Scope

* Health questionnaire content
* Email templates
* Billing or invoicing logic

---

## 4. High-Level Architecture

```
Order Created
     │
     ▼
Order Type Selected (Dropship | Kit On-site)
     │
     ├─ Dropship
     │     ▼
     │  Auto Registration Processor
     │
     └─ Kit On-site
           ▼
     Manual Registration (Practitioner or Client)
```

Visibility enforcement and audit updates occur at the point of registration.

---

## 5. Registration Pathways

A kit may be registered via one of three paths:

### 1. Automatic Registration

* Triggered for dropship orders
* Performed by background processors
* Practitioner involvement not required

### 2. Manual Practitioner Registration

* Used for on-site clinic workflows
* Initiated by practitioners
* Explicitly marked as manual

### 3. Client Self-Registration

* Performed via the client portal
* Requires visibility reconciliation when kits were purchased by a practitioner

All paths converge on a shared audit and visibility model.

---

## 6. Registration Source Attribution

To ensure accurate auditability, practitioner registration flows explicitly declare how registration occurred.

### RegistrationSource Enum

* `AUTO`
* `MANUAL`

Usage:

* Automatic registration processors pass `AUTO`
* Practitioner registration passes `MANUAL`
* Used to set `PractitionerKit.registeredViaAuto`

Client self-registration does not use this enum.

---

## 7. Order-Kit Registration Audit

`order-kits` is the authoritative record for kit registration state when a kit is tied to an order.

On successful registration, the corresponding `order-kits` record must be updated with:

* `registrationStatus = YES`
* `registeredBy = PRACTITIONER | CLIENT`
* `registeredByUserId = <userId>`

This ensures:

* Clear per-kit audit history
* Reliable debugging and reporting
* Accurate downstream reasoning

`orders.kitId` is ignored entirely.

---

## 8. Practitioner Visibility for Client-Registered Kits

### Problem

Clients can register kits themselves. When the kit was purchased by a practitioner, the practitioner must retain access to results without requiring manual approval from the client.

### Required Behavior

When a client registers a kit:

1. Attempt to locate an `order-kits` record using `kitId`
2. If no record exists:

   * Proceed normally
   * Do not modify practitioner access
3. If a record exists:

   * Join to `orders` to retrieve `order.userId` (practitioner user)
   * Fetch `practitioner.id`
   * Ensure a `client_practitioner` record exists with `reportAccess = GRANTED`

Rules:

* Create record if missing
* Upgrade access if declined
* No-op if already granted
* Multiple practitioners may have access

This guarantees practitioner visibility for all practitioner-purchased kits.

---

## 9. Query Strategy (Optimized)

Visibility enforcement is implemented using two queries:

1. Fetch `OrderKit` joined to `Order` to retrieve:

   * `orderId`
   * `order.userId`
2. Fetch `Practitioner` by `userId` (select `id` only)

This balances performance with clarity.

---

## 10. Practitioner Platform UX Requirement

### Navigation Clarity

The practitioner side navigation item must be renamed:

* **From:** Register Client Kit
* **To:** **Register On-Site Kits**

### Intent Communication

The page must clearly state that:

* This section is only for kits handed to patients in person
* Dropshipped kits are handled automatically

This removes ambiguity and prevents incorrect manual actions.

---

## 11. Implementation Checklist (Engineering)

### Backend

* Add `RegistrationSource` enum
* Update practitioner registration service to accept source
* Update `order-kits` audit fields on all registration paths
* Implement practitioner visibility auto-grant logic
* Ensure idempotency and logging

### Frontend

* Rename practitioner navigation item
* Add explanatory copy to manual registration page

---

## 12. Acceptance Criteria

* Dropship kits auto-register without manual intervention
* On-site kits remain available for manual registration
* Practitioner platform clearly communicates when action is required
* Client-registered practitioner kits automatically grant practitioner access
* Order-kit audit fields are consistently populated
* No duplicate visibility records are created
* No support intervention required for expected workflows

---

## 13. Conditional Client Identity Collection (Delivery Mode–Aware)

### Background

During practitioner order creation, client identity fields were previously collected unconditionally. This does not align with real-world **Kit On-Site** workflows, where kits are shipped to clinics and later handed to patients whose identities are not yet known.

To prevent invalid data capture, confusion, and downstream inconsistencies, client identity collection must now be **explicitly tied to delivery mode**.

---

### Delivery Mode Rules

#### Kit On-Site (`deliveryMode = ON_SITE`)

When **Kit On-Site** is selected:

* Client **first name**, **last name**, and **email** must **not** be collected
* These fields must be:

  * Hidden in the UI
  * Optional at the DTO level
  * Persisted as `NULL` / `undefined` in the database
* Empty strings submitted from the frontend must be transformed to `undefined`

Rationale:
Kits are not associated with a specific client at order time. Client assignment occurs later during manual or client-driven registration.

---

#### Dropship (`deliveryMode = DROPSHIP`)

When **Dropship** is selected:

* Client **first name**, **last name**, and **email** are **required**
* These fields must:

  * Be visible in the UI
  * Be strictly validated (non-empty, non-null)
  * Be rejected if missing or invalid

Rationale:
Dropship orders are shipped directly to a known client and rely on client identity for fulfillment and downstream automation.

---

### Backend Validation Requirements

Validation must be enforced at **multiple layers** to ensure correctness and fail fast:

#### DTO-Level Validation

* Client identity fields (`firstName`, `lastName`, `email`) are:

  * Required **only if** `deliveryMode !== ON_SITE`
  * Optional **only if** `deliveryMode === ON_SITE`
* Payload transformation must convert empty strings to `undefined` before validation

This prevents accidental persistence of invalid placeholder values.

---

#### Service-Level Guardrails

In addition to DTO validation, **order creation services must explicitly enforce delivery mode rules**:

* If `deliveryMode === DROPSHIP`:

  * Reject orders missing client first name, last name, or email
* If `deliveryMode === ON_SITE`:

  * Ignore client identity fields entirely
  * Do not persist them on the `orders` table

This ensures protection against:

* Bypassed DTO validation
* Programmatic misuse
* Future regressions

---

### Data Persistence Rules

On the `orders` table:

* For **Kit On-Site** orders:

  * Client identity fields must remain `NULL`
* For **Dropship** orders:

  * Client identity fields must be fully populated and valid

This maintains data integrity and aligns storage with real-world intent.

---

### Outcome

This update ensures:

* No premature or invalid client data is collected
* Order intent is unambiguous
* Dropship automation remains reliable
* On-site clinic workflows remain clean and flexible
* Backend enforcement matches UI behavior
* Support and audit complexity is reduced


### DTO Enforcement Example (Illustrative)

The order creation DTO must enforce conditional client identity validation based on `deliveryMode`.

```ts
export enum DeliveryMode {
  DROPSHIP = 'dropship',
  ON_SITE = 'on_site',
}

const emptyStringToUndefined = ({ value }: { value: unknown }) =>
  typeof value === 'string' && value.trim() === '' ? undefined : value;

  @ApiPropertyOptional({ enum: DeliveryMode })
  @IsEnum(DeliveryMode)
  @IsOptional()
  deliveryMode?: DeliveryMode;

  @ApiPropertyOptional()
  @Transform(emptyStringToUndefined)
  @ValidateIf((o) => o.deliveryMode !== DeliveryMode.ON_SITE)
  @IsString()
  @IsNotEmpty()
  firstName?: string;

  @ApiPropertyOptional()
  @Transform(emptyStringToUndefined)
  @ValidateIf((o) => o.deliveryMode !== DeliveryMode.ON_SITE)
  @IsString()
  @IsNotEmpty()
  lastName?: string;

  @ApiPropertyOptional()
  @Transform(emptyStringToUndefined)
  @ValidateIf((o) => o.deliveryMode !== DeliveryMode.ON_SITE)
  @IsEmail()
  @IsNotEmpty()
  email?: string;
```


### Enforcement Notes

* DTO validation **does not replace** service-level checks
* Order creation services must still explicitly enforce:

  * Client identity is **required** for `DROPSHIP`
  * Client identity is **ignored** for `ON_SITE`

This layered approach guarantees correctness even if validation is bypassed.


## Implementation Checklist (Engineering)



1. Add `RegistrationSource` enum (`AUTO`, `MANUAL`) and wire it into `RegisterPractitionerKitService`.

2. Set `PractitionerKit.registeredViaAuto` based on `RegistrationSource`.

3. Introduce `registeredBy` enum (`PRACTITIONER`, `CLIENT`) on `order-kits`.

4. Add `registeredByUserId` column to `order-kits`.

5. On **any successful kit registration**, update `order-kits`:



   * `registrationStatus = YES`

   * `registeredBy`

   * `registeredByUserId`

6. Ensure practitioner manual registration sets:



   * `registeredBy = PRACTITIONER`

   * `registeredByUserId = practitioner.userId`

7. Ensure client self-registration sets:



   * `registeredBy = CLIENT`

   * `registeredByUserId = client.userId`

8. In client registration flow, locate `order-kits` by `kitId` and join to `orders` to retrieve practitioner `userId`.

9. Resolve `practitioner.id` using `userId` (select `id` only).

10. Auto-grant practitioner access by upserting `client_practitioner` (`GRANTED` if missing or declined).

11. Allow multiple practitioners per client; prevent duplicate access rows.

12. Ignore `orders.kitId` entirely and rely exclusively on `order-kits`.

13. Rename practitioner side navigation item to **Register On-Site Kits** and add clarifying copy.

14. Ensure all flows are idempotent and log auto-grant events for traceability.

15. Enforce delivery-mode–aware client identity handling by requiring client name and email only for DROPSHIP orders and omitting them entirely for ON-SITE orders, with UI hiding/clearing, DTO-level transform + conditional validation, and service-level guards to prevent invalid persistence.
