# Executive Summary PRD

**Automated Practitioner Kit Registration & Clinic Workflow Completion**

## Purpose

This document summarizes the full scope of changes required to complete the **Automated Practitioner Kit Registration** initiative, while correctly supporting **clinic (on-site) workflows**, **manual registrations**, and **practitioner visibility**.

The goal is to confirm that the proposed behavior and product decisions are acceptable **before proceeding to a detailed technical PRD and implementation work**.

---

## Background

Automated kit registration has been introduced for practitioner orders:

* Kits shipped **directly to patients** are automatically registered
* Health questionnaires are sent automatically after registration
* Manual registration remains available for on-site use cases

This significantly reduces operational overhead and improves reliability for the majority of orders.

As automation was introduced, several **real-world clinic and manual registration edge cases** became visible that require explicit handling to avoid workflow blockers and visibility issues.

---

## Problem Statement 1: On-Site Clinic Orders vs Dropship Orders

### Current Issue

Some clinics place **bulk orders** and ship kits **to their clinic**, not directly to patients.

In these cases:

* Kits are stocked on-site
* Kits are handed to patients in person
* Kits are registered manually at the point of handoff

With automatic registration enabled, these on-site kits can be registered too early, which prevents clinics from registering them correctly for patients.

---

### Proposed Change

Introduce an explicit **delivery mode selection** at order time, presented as a **toggle**, before client email and address fields.

**Delivery modes:**

* **On-site handoff at clinic**

  * Kits are handed to patients in person
  * Kits are **not auto-registered**
  * Manual registration is required

* **Dropship to patient**

  * Kits are shipped directly to patients
  * Kits are auto-registered
  * Health questionnaires are sent automatically

This makes clinic intent explicit and ensures the correct workflow is applied.

---

## Problem Statement 2: Clarity Around Manual Registration in Practitioner Platform

### Current Issue

With automation in place, the existing practitioner side navigation item:

* **Register Client Kit**

is ambiguous and may cause confusion about whether practitioners still need to manually register kits.

---

### Proposed Change

Clarify intent through navigation and copy updates:

* Rename side navigation item:

  * From: **Register Client Kit**
  * To: **Register On-Site Kits**
* Add explanatory text indicating this page is only for kits handed to patients in person at clinics

This clearly separates:

* Automated dropship workflows
* Manual on-site workflows

---

## Problem Statement 3: Practitioner Visibility for Client-Registered Kits

### Current Issue

There is an existing flow where:

* Clients can create accounts and register kits themselves
* Practitioners can only see kits and reports **if the client explicitly grants access**

In practice:

* Some clients forget or do not know how to grant access
* Practitioners who ordered the kits cannot see kits or reports
* Manual support intervention is required

---

### Proposed Change

When a kit is **ordered by a practitioner**, the system should automatically grant that practitioner access to the client account **if the client performs the registration themselves**.

This ensures:

* Practitioner-ordered kits are always visible to the practitioner
* Manual access-granting by the client is no longer required in this scenario

This applies only when the practitioner is the original purchaser of the kit.

---

## Problem Statement 4: Tracking On-Site Kit Registrations

### Current Issue

On-site kit registrations:

* Occur manually
* Are not tracked separately in a dedicated structure
* Lack a clear foundation for future on-site programs

---

### Proposed Change

Introduce a new internal system table to record **on-site kit registration events**.

For now, this table will:

* Record all on-site registrations
* Be non-user-facing
* Remain reserved for future use (e.g., on-site payment programs, reporting)

This provides a clean foundation without introducing immediate user-facing changes.

---

## Scope Notes

* No changes to laboratory workflows
* No changes to questionnaire content
* No changes to existing automated dropship flows
* No removal of client self-registration
* Manual health questionnaire sending already exists and is unchanged

---

## Summary

With automated registration in place, the system must now correctly support:

* Both dropship and on-site clinic workflows
* Manual and client-driven registration paths
* Guaranteed practitioner visibility into kits they ordered
* A future-proof structure for on-site programs

This single set of changes completes the automation effort while ensuring correctness, clarity, and scalability as both automated and manual workflows coexist.
