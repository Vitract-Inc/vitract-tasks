# **Unified Kits Migration Runbook**

**Kit + PractitionerKit Consolidation (v3)**
**Status:** Approved design, implementation-ready
**Audience:** Engineering, Platform Ops, Product, Leadership

---

## 1. Executive Summary (Non-Technical)

### What we’re doing

We are consolidating two separate kit records (`kit` and `practitioner_kits`) into **one unified system** called **`kits`**.

### Why this matters

Today, kits exist in two places depending on who owns them. This creates:

* duplicated logic,
* inconsistent behavior,
* higher maintenance cost,
* and risk when changing or scaling the system.

By moving to **one unified kits table**, we simplify the system while **keeping all existing data safe**.

### What will *not* change

* No user-facing behavior changes
* No practitioner workflow changes
* No data is deleted
* No downtime required

### How risk is controlled

* Old tables stay **read-only** (for rollback and audit)
* Migration is **command-driven**, reversible, and logged
* Background jobs are paused during cutover
* Rollback can be executed per migration batch

### End result

* One source of truth for all kits
* Cleaner services, queues, and APIs
* Safer future features and migrations

---

## 2. High-Level Strategy (At a Glance)

| Area        | Decision                                      |
| ----------- | --------------------------------------------- |
| Data model  | New `kits` table (single source of truth)     |
| Legacy data | Preserved, read-only                          |
| Ownership   | Explicit `ownerType` + `practitionerId` rules |
| Migration   | Command-driven, idempotent, batch-tracked     |
| Rollback    | Deterministic per batch                       |
| Downtime    | None (zero-downtime cutover)                  |

---

## 3. Implementation Phases Overview

This migration is executed in **7 controlled phases**.
Each phase has a **clear entry condition**, **exit condition**, and **rollback boundary**.

```
Phase 0 → Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7
Analysis   Schema     Services   APIs       Queues     Modules    Validation Cleanup
```

---

## 4. Phase-by-Phase Runbook

---

## Phase 0 — Pre-Migration Analysis (GO / NO-GO)

### Purpose (Plain Language)

Before touching anything, we confirm the data is safe to migrate and that ownership rules are consistent.

### What happens

* Validate practitioner → user mappings
* Detect duplicate or invalid kit numbers
* Identify hard blockers that would cause incorrect merges

### Owners

Engineering + DB + Platform Ops

### Go / No-Go Conditions

❌ **Migration must NOT proceed if any blocker exists**

**Hard blockers include:**

* Practitioners without a user account
* Multiple practitioners sharing one user
* Duplicate `kitNumber` across or within tables
* Missing or empty `kitNumber`

### Output

* Signed-off validation report
* Explicit approval to proceed

---

## Phase 1 — Schema & Entity Creation

### Purpose

Create the new unified structure **without affecting production traffic**.

### What happens

* Create new `kits` table
* Create migration tracking tables
* Introduce new `Kits` entity in code
* No reads or writes are switched yet

### Why this is safe

* No legacy tables are modified
* No application logic is changed

### Rollback

* Drop newly created tables only (no data loss)

---

## Phase 2 — Data Migration (Command-Driven)

### Purpose

Copy all legacy kit data into `kits` in a **controlled, auditable way**.

### What happens

* Workers are paused
* Migration command runs:

  * generates new kit IDs
  * maps legacy IDs → new IDs
  * records provenance (`batchId`, timestamps, source)
* No application traffic is affected

### Key Rules

* Migration is **idempotent**
* Duplicate kit numbers **fail the migration**
* Every run produces a batch record

### Output

* `kits` populated
* `kit_id_map` created
* Migration batch logged

### Rollback

* Delete by `migrationBatchId`
* Legacy tables remain untouched

---

## Phase 3 — Service Layer Switch

### Purpose

Ensure **all business logic** uses the unified kits model.

### What happens

* Services stop reading/writing legacy tables
* All logic targets `Kits`
* Ownership rules enforced in one place

### Optional Safety Net

* **Dual-write window** (1 deploy):

  * Write to `kits` + legacy
  * Read only from `kits`
  * Legacy failures are non-fatal
  * `kits` failures are fatal

### Rollback

* Toggle feature flag
* Services fall back to legacy tables

---

## Phase 4 — Controllers & API Layer

### Purpose

Keep API behavior identical while switching the backing store.

### What happens

* Controllers call unified services
* DTOs are consolidated
* Responses include explicit ownership context

### Client Impact

* None (contracts preserved)
* Deprecations announced but not enforced

### Rollback

* Re-enable legacy service bindings

---

## Phase 5 — Queues & Background Jobs

### Purpose

Prevent async workflows from breaking during cutover.

### What happens

* Processors read/write `kits` only
* Payloads accept:

  * `kitsId` (preferred)
  * legacy IDs (mapped)
  * `kitNumber` (last resort)
* Workers paused during migration window

### Deterministic Payload Resolution

1. Use `kitsId`
2. Else map legacy ID
3. Else lookup by `kitNumber`

### Rollback

* Re-enable legacy payload decoding

---

## Phase 6 — Testing & Validation

### Purpose

Prove the system behaves exactly the same — or better.

### Required Checks

* Unit tests for all modified services
* Integration tests for practitioner & client flows
* Data integrity validation:

  * counts
  * ownership correctness
  * no duplicate kit numbers

### Acceptance Criteria

* Unified results match legacy behavior
* Events and queues fire correctly
* Rollback tested successfully

---

## Phase 7 — Cleanup & Deprecation

### Purpose

Stabilize and document the new system.

### What happens

* Legacy tables remain read-only
* Legacy entities removed from active code paths
* Documentation updated
* Monitoring period begins

### What does *not* happen

* No table drops
* No irreversible deletes

---

## 5. Cutover Checklist (Operator Runbook)

**In order, no skipping steps:**

1. Deploy schema + feature flags
2. Pause workers
3. Enable migration guard (`MIGRATION_IN_PROGRESS=true`)
4. Run migration command
5. Validate counts and mappings
6. Enable unified reads
7. Enable unified writes
8. Resume workers
9. Monitor for 24–48 hours

---

## 6. Rollback Strategy (Guaranteed)

Rollback is **batch-scoped** and **deterministic**.

If anything goes wrong:

* Disable unified flags
* Delete `kits` rows by `migrationBatchId`
* Resume legacy reads/writes
* No data loss, no corruption

---

## 7. Final Outcome

After completion:

* One clean, authoritative `kits` model
* No duplicated logic
* Safer feature development
* Strong audit and rollback guarantees
