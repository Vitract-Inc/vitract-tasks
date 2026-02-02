# **Valid Kits Upload & Management System**

---

## 1. Executive Summary

The **Valid Kits Upload & Management System** is a core internal ingestion pipeline within **`vitract-kit-api-v1`**, responsible for accepting, validating, and persisting provider-supplied kit identifiers into the Vitract ecosystem.

These kits act as the **canonical source of truth** for:

* Determining whether a kit is valid
* Identifying the test type associated with a kit
* Grouping multiple physical samples under a single logical test (e.g. DeepGut Plus)

The current implementation is fragile, partially unverified, and does not support modern operational requirements such as transactional safety, multi-sample kits, deterministic validation, or internal observability.

This PRD defines a **robust, transactional, extensible, and auditable** system that:

* Accepts CSV uploads from providers
* Performs strict schema and business validation
* Supports primary and related kit identifiers
* Derives and enforces kit test types
* Uses safe, atomic bulk inserts
* Exposes a fully usable **internal Swagger interface**
* Provides a clean path for historical data reconciliation

---

## 2. Problem Statement

### Current State Issues

1. **Unreliable bulk ingestion**

   * Partial writes possible
   * No atomicity guarantees
   * Risk of inconsistent datasets

2. **Incomplete data modeling**

   * Only one kit ID supported
   * Cannot represent multi-sample tests
   * No persisted test type

3. **Weak validation pipeline**

   * CSV schema loosely enforced
   * Business rules inconsistently applied
   * Error reporting lacks row-level clarity

4. **Performance & safety concerns**

   * Repository `.save()` loops
   * Risk of N+1 behavior
   * No explicit transaction boundaries

5. **Poor internal developer experience**

   * Swagger does not support file upload properly
   * No clear ingestion contract
   * Hard to debug or rerun safely

---

## 3. Goals & Non-Goals

### Goals

* Safe, repeatable ingestion of provider kit data
* Full support for **multi-sample kits**
* Strict validation of kit formats and prefixes
* Zero partial writes on failure
* Deterministic and observable ingestion behavior
* Internal Swagger usability for operations teams

### Non-Goals

* Public or external upload endpoints
* Provider-facing UI
* Streaming or real-time ingestion
* Arbitrary schema uploads

---

## 4. System Ownership & Scope

**Authoritative API:**

> **`vitract-kit-api-v1`**

All changes described in this PRD are implemented **exclusively** in this service.

Scope includes:

* Entity definitions
* Validation utilities
* Bulk ingestion service
* Controller & Swagger docs
* Backfill command

---

## 5. High-Level Architecture

```
CSV Upload
   ↓
File Parsing (csvtojson)
   ↓
Schema Validation (DTO + class-validator)
   ↓
Business Validation (kit prefixes, duplicates)
   ↓
Normalization (dates, casing, trimming)
   ↓
Test Type Derivation
   ↓
Transactional Bulk Insert
   ↓
Structured Logging & Metrics
```

---

## 6. Data Model Changes

### 6.1 ValidKit Entity (Updated)

#### Naming Decision (Intentional)

* `kitId` is the **primary / canonical identifier**
* Additional kit IDs represent **related samples**
* Flat columns are preferred over JSON for:

  * Indexing
  * Validation
  * Operational clarity

### Updated Entity Definition

```ts
@Entity('valid-kit')
export class ValidKit extends BaseEntity {
  @Column({ type: 'varchar' })
  status: string;

  @Column({ type: 'varchar', nullable: true })
  expiryDate: string;

  @Column({ type: 'varchar', nullable: true })
  sampleType: string;

  @Column({ type: 'varchar', nullable: true })
  sequenceType: string;

  /** Primary / canonical kit identifier */
  @Column({ type: 'varchar', unique: true })
  kitId: string;

  /** Related identifiers for multi-sample kits */
  @Column({ type: 'varchar', nullable: true })
  relatedKitId1: string;

  @Column({ type: 'varchar', nullable: true })
  relatedKitId2: string;

  /** Derived test classification */
  @Column({ type: 'varchar' })
  testType: string;
}
```

---

## 7. Kit Type Detection & Enforcement

### Source of Truth

Kit type is **derived**, never user-supplied.

```ts
detectKitTypeFromNumber(kitId)
```

### Prefix Rules

| Prefix  | Test Type   |
| ------- | ----------- |
| VT1–VT4 | GutScan     |
| 00      | DeepGut     |
| DP      | DeepGutPlus |

### Enforcement Rules

* `testType` **must not** be provided in CSV
* Derived test type **must match prefix**
* Invalid prefixes → hard failure

---

## 8. CSV Upload Requirements

### Supported Formats

* CSV (mandatory)
* Excel `.xlsx` (future)

### Expected Columns

| Column        | Required | Notes              |
| ------------- | -------- | ------------------ |
| kitId         | Yes      | Primary identifier |
| relatedKitId1 | No       | Optional           |
| relatedKitId2 | No       | Optional           |
| expiryDate    | No       | ISO-parsable       |
| sampleType    | No       | Free text          |
| sequenceType  | No       | Free text          |

### Validation Rules

* `kitId`:

  * Required
  * Unique within file
  * Unique in DB
  * Valid prefix

* Related Kit IDs:

  * Cannot equal `kitId`
  * Cannot duplicate each other
  * Must have valid prefixes

* Dates:

  * Normalized to ISO
  * Invalid formats rejected

---

## 9. Bulk Insert Strategy (Authoritative)

### Transaction Model (Non-Negotiable)

All writes MUST be wrapped using:

```ts
this.dataSource.transaction(async (manager) => {
  // writes
});
```

❌ Manual `QueryRunner`
❌ `repository.save()` in loops
❌ Partial batch commits

---

### Required Insert Pattern

```ts
await this.dataSource.transaction(async (manager) => {
  for (let i = 0; i < kits.length; i += BATCH_SIZE) {
    const batch = kits.slice(i, i + BATCH_SIZE);

    await manager
      .createQueryBuilder()
      .insert()
      .into(ValidKit)
      .values(batch)
      .execute();
  }
});
```

### Guarantees

* Atomic ingestion
* Automatic rollback on any error
* Safe batching for large datasets
* Deterministic failure behavior

---

## 10. API Design

### Endpoint (Internal Only)

```
POST /api/v1/valid-kits/bulk
```

### Request

* `multipart/form-data`
* Field: `csvFile`

### Success Response

```json
{
  "status": "success",
  "message": "Valid kits uploaded successfully",
  "data": {
    "totalProcessed": 1200,
    "inserted": 1200
  }
}
```

### Failure Response

* Row-specific validation errors
* Column-level error messages
* No partial inserts

---

## 11. Swagger Requirements (Internal)

* Available only on **internal Swagger**
* Supports file upload
* Documents:

  * CSV schema
  * Validation rules
  * Failure examples

```ts
@ApiConsumes('multipart/form-data')
@ApiBody({
  schema: {
    type: 'object',
    properties: {
      csvFile: {
        type: 'string',
        format: 'binary',
      },
    },
  },
})
```

---

## 12. Historical Data Reconciliation

### Decision: CLI Command (Not Migration)

**Rationale:**

* Business logic involved
* Safe to rerun
* Observable
* Production-friendly

### Command Responsibilities

* Iterate existing `valid-kit` records
* Infer `testType` via `detectKitTypeFromNumber`
* Populate missing or incorrect values
* Run inside a transaction
* Log summary metrics

### Execution

```bash
RUN_COMMAND=true node dist/main backfill:valid-kit-test-types
```

---

## 13. Observability & Safety

### Logging

* Upload start/end
* Total rows processed
* Batch timings
* Error summaries

### Error Handling

* Fail fast
* Structured validation errors
* No silent skips

---

## 14. Open Questions (Tracked)

1. Will any test require **more than 3 related kit IDs**?
2. Should related kit IDs be globally unique?
3. Will provider attribution be required later?

---

## 15. Success Metrics

* Zero partial uploads
* Deterministic error reporting
* <1s validation for ≤1k rows
* Re-runnable backfill without side effects

---

## Final Notes (Explicit)

* This PRD **supersedes** all prior implementations
* Current code is **not production-grade**
* Transactional integrity is the core improvement
* Schema simplicity is intentional and correct
