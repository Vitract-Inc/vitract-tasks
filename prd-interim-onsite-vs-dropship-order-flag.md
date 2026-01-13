# Interim PRD: On-Site vs Dropship Order Flag

*(Pre-Full Implementation Update)*

## Purpose

This PRD defines a **short-term, targeted update** required to safely support **on-site clinic orders** ahead of the full Automated Kit Registration implementation.

It introduces a minimal mechanism to distinguish **on-site** orders from **dropship** orders so that auto-registration can be conditionally disabled where it does not apply.

This document intentionally covers **only what is needed immediately**.
All broader workflow, access, and tracking improvements will be addressed in the **full PRD**.

---

## Background

Automated kit registration currently applies to all practitioner orders once shipped.

However, some practitioners place orders for **on-site clinic use**, where kits are:

* Shipped to the clinic
* Handed to patients in person
* Registered manually at the point of handoff

Because these orders are not currently distinguishable, on-site kits may be auto-registered incorrectly.

---

## Scope of This Interim PRD

This update is limited to:

* A **single UI control** on the order purchase form
* A **single new field** on the order record
* A **conditional skip** in the auto-registration flow

Nothing else.

---

## Proposed Changes

## 1. Frontend: Order Purchase Form Update

Add a **delivery-mode toggle** to the order purchase form.

### Intent

To explicitly capture whether an order is:

* For **on-site clinic handoff**
* Or **dropshipped directly to a client**

### Toggle Copy (Initial)

**Toggle label:**

> **On-site kit handoff at clinic**

**Helper text:**

> Enable this if you will be handing these kits directly to patients at your clinic.
> On-site kits must be registered manually after handoff.
>
> Leave this off if kits will be shipped directly to patients — dropshipped kits are auto-registered under your practitioner account.

> **Note:**
> This copy is approved for the interim implementation and may be refined later by design, provided the behavior remains unchanged.

### UI Notes

* Toggle placement and visual treatment will be handled by **Tosin**
* Toggle should integrate naturally into the order form
* Placement should be **before** client email and shipping address fields

### Behavior

* Toggle **ON** → On-site clinic order
* Toggle **OFF** → Dropship order (default)

---

## 2. Backend: Order Data Model Update

Add a field to the `orders` table representing delivery mode.

### Field Definition (Confirmed)

Use an **ENUM** with the following values:

* `DROPSHIP`
* `ON_SITE`

### Behavior

* Field is set at order creation
* Defaults to `DROPSHIP` for backward compatibility
* Persists the practitioner’s intent at purchase time

---

## 3. Auto-Registration Logic (Stopgap Rule)

Update auto-registration behavior to respect the delivery mode.

### Rule

* If `delivery_mode = ON_SITE`
  → **Skip auto-registration entirely**
* If `delivery_mode = DROPSHIP`
  → Existing auto-registration logic applies unchanged

No additional branching or side effects are required for this interim update.

---

## Execution Plan (Immediate)

This work is expected to proceed **quickly and sequentially**:

1. **Tosin**

   * Finalizes toggle placement and UI indicators on the order form
   * Ensures the control does not appear out of place

2. **Nengak**

   * Begins backend work in parallel
   * Implements enum field and default behavior
   * Wires frontend toggle to backend field
   * Applies conditional auto-registration logic once UI is finalized

This work should start **immediately** and is intended to unblock operations while the full PRD is prepared.

---

## Out of Scope (Explicit)

The following are **not included** in this interim PRD:

* Practitioner visibility fixes for client-registered kits
* On-site registration tracking tables
* Navigation or copy changes outside the order form
* Access-granting logic
* Payment or reporting logic
* Questionnaire changes

All of the above will be addressed in the **full implementation PRD**.
