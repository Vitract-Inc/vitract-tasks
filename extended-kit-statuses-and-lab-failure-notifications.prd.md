# Extended Kit Statuses, Notes, and Lab Failure Notifications

---

## 1. Executive Summary

This PRD defines enhancements to the kit lifecycle to accurately represent lab failure and retry scenarios while improving communication with customers and practitioners.

Three new kit statuses—**sample failed**, **cannot be processed**, and **re-sequencing**—will be introduced. These statuses require explanatory notes, trigger dedicated email notifications, and, in the case of re-sequencing, automatically adjust the expected result delivery date.

The update improves transparency, reduces support friction, and creates a structured audit trail for lab-related exceptions.

---

## 2. Problem Statement

The current kit lifecycle does not capture real-world laboratory failure scenarios, resulting in unclear customer communication, manual support intervention, and lack of traceability for why a kit did not progress normally.

---

## 3. Goals & Non-Goals

### Goals

* Accurately model lab failure and retry states
* Require contextual notes for failure-related statuses
* Notify customers and practitioners clearly and consistently
* Adjust expected result timelines automatically for re-sequencing
* Preserve an audit trail of lab exceptions

### Non-Goals

* No changes to lab processing or sequencing logic
* No redesign of report generation
* No public API exposure of notes
* No event architecture redesign

---

## 4. Kit Status Model

### 4.1 New Statuses (Additive)

The `KitStatus` enum will be extended with:

* `sample-failed`
* `cannot-be-processed`
* `re-sequencing`

These are **kit-level statuses** and coexist with existing statuses such as `lab-processing` and `result-ready`.

---

### 4.2 Status Semantics

| Status              | Terminal | Description                                |
| ------------------- | -------- | ------------------------------------------ |
| sample-failed       | Yes      | Sample failed QC checks and cannot proceed |
| cannot-be-processed | Yes      | Sample is unusable due to lab constraints  |
| re-sequencing       | No       | Sample is being reprocessed                |

---

### 4.3 Re-sequencing Date Adjustment

* When status becomes `re-sequencing`, update `resultsAvailable` to **current date + 14 days**
* If re-sequencing occurs again, reset the date to **current date + 14 days**
* Dates are not cumulative

---

## 5. Notes Requirement

### 5.1 Purpose

Notes provide human-readable explanations for lab failures or retries. They are included in outbound emails and stored for internal traceability.

---

### 5.2 Validation Rules

Notes are **required** when status is one of:

* `sample-failed`
* `cannot-be-processed`
* `re-sequencing`

Notes are optional for all other statuses.

If notes are passed as an empty string, email content must instruct the recipient to contact support for further information.

---

### 5.3 Notes Persistence Model (Entity)

Notes will be stored in a separate table using the following entity:

```ts
import { BaseEntity } from 'src/common';
import { Column, Entity, Index, JoinColumn, ManyToOne } from 'typeorm';
import { Kit } from './kit.entity';

@Index('ix_kit_status_notes_kit_id_created_at', ['kitId', 'createdAt'])
@Index('ix_kit_status_notes_kit_number_created_at', ['kitNumber', 'createdAt'])
@Entity('kit_status_notes')
export class KitStatusNote extends BaseEntity {
  @Column({ type: 'uuid' })
  kitId: string;

  @Column({ type: 'varchar' })
  kitNumber: string;

  @Column({ type: 'varchar' })
  status: string;

  @Column({ type: 'nvarchar', length: 'max' })
  notes: string;

  @Column({ type: 'varchar', nullable: true })
  createdBy?: string;

  @ManyToOne(() => Kit, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'kitId' })
  kit: Kit;
}
```

**Design Characteristics**

* One note per status change
* Multiple notes allowed per kit
* Immutable records (insert-only)
* Optimized for audit and support queries

---

## 6. API Changes

### 6.1 DTO Extension

The existing `UpdateKitStatusDto` will be extended with:

* `notes?: string`

This applies uniformly to:

* Internal kit status updates
* External API status updates
* Practitioner kit status updates

No new endpoints are required.

---

### 6.2 Backward Compatibility

* Existing clients remain unaffected
* Validation is enforced only for the three new statuses
* Status updates without notes fail only when notes are required

---

## 7. Email Notifications

### 7.1 Trigger Conditions

Emails are sent when a kit transitions into:

* `sample-failed`
* `cannot-be-processed`
* `re-sequencing`

---

### 7.2 Recipient Rules

* **Client:** Always notified
* **Practitioner:**

  * Notified if the client granted report access
  * Notified exclusively if the kit was registered by a practitioner
* If no practitioner access exists, only the client receives the email

---

### 7.3 Sender & CC

* **From:** `support@vitract.com`
* **CC:**

  * `venice@vitract.com`
  * `info@vitract.com`

Replies must route back to support.

---

### 7.4 Email Content Requirements

Each email must include:

* Kit ID
* Current status
* Notes content (or fallback instruction)
* Recipient name
* Updated expected result date (for re-sequencing)
* CTA: **View Reports**

---

### 7.5 Email Templates

Three dedicated templates:

1. Sample Failed
2. Cannot Be Processed
3. Re-sequencing

Templates provided by **Habeeb** are implemented strictly within this scope and priority.

---

## 8. Frontend Requirements

* New statuses must be visually distinct
* Each new status must have a dedicated badge color
* Badge color definitions are owned by **Tosin (Product Designer)**
* **Habeeb** will implement Tosin’s designs on the frontend

---

## 9. Ownership & Responsibilities

### 9.1 Engineering – **Habeeb**

* Extend `KitStatus` enum
* Add notes validation to DTOs
* Implement notes persistence
* Apply re-sequencing date logic
* Update status update services
* Implement email triggers and payloads
* Implement frontend status badges per design

---

### 9.2 Product Design – **Tosin**

* Define badge colors for new statuses
* Ensure consistency with existing design system
* Deliver finalized designs to engineering

---

## 10. Checklists

### 10.1 Habeeb – Engineering Checklist

* [ ] Extend `KitStatus` enum
* [ ] Add `notes` to `UpdateKitStatusDto`
* [ ] Enforce notes validation
* [ ] Create `KitStatusNote` entity
* [ ] Persist one note per status change
* [ ] Implement re-sequencing date adjustment
* [ ] Update all status update services
* [ ] Implement email templates and routing
* [ ] Implement frontend badge rendering
* [ ] Verify backward compatibility

---

### 10.2 Tosin – Product Design Checklist

* [ ] Define badge colors for new statuses
* [ ] Validate contrast and accessibility
* [ ] Share final specs with Habeeb

---
