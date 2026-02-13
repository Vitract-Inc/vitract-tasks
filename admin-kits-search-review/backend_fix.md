# 📘 Admin Kits Search — Enterprise Full-Text Search Upgrade (MSSQL)

# 1. Executive Summary

The current admin kits search uses `LIKE '%query%'` across multiple columns with OR conditions. This approach:

* Does not scale
* Prevents effective index usage
* Performs full scans at moderate dataset sizes
* Has unpredictable performance
* Is not enterprise-grade

This document defines the migration to **MSSQL Full-Text Search (FTS)** to deliver:

* Scalable, indexed search
* Multi-word token support
* Full name search (`firstName + lastName`)
* Deterministic, secure, predictable behavior
* Production-safe migration strategy

---

# 2. Problem Statement

Current search:

* Uses `%query%` across:
  * `kits.kitNumber`
  * `kits.name`
  * `user.firstName`
  * `user.lastName`
  * `familyKit.kitNumber`
  * `familyKit.name`
* Applies OR logic
* No input normalization
* No explicit collation

At scale, this leads to:

* Table scans
* Slow pagination
* CPU spikes
* Performance instability

---

# 3. Goals

### Functional Goals

1. Replace `LIKE`-based search with MSSQL Full-Text Search.
2. Support fullName search:
   * `"john doe"`
   * `"doe john"`
   * `"john"`
3. Maintain existing type filtering (`FAMILY`, `PRACTITIONER`, `CLIENTS`).
4. Maintain deterministic ordering.
5. Preserve current API contract.

### Non-Functional Goals

1. Index-backed search.
2. Stable performance at 1M+ rows.
3. Case-insensitive and accent-insensitive.
4. Secure against injection.
5. Production-safe migration.

---

# 4. Non-Goals

* No introduction of external search engines.
* No fuzzy ranking beyond FTS defaults.
* No change to response structure.
* No change to pagination semantics.

---

# 5. High-Level Architecture

## Current

```
WHERE (
   kitNumber LIKE '%q%'
   OR name LIKE '%q%'
   OR firstName LIKE '%q%'
   OR lastName LIKE '%q%'
)
```

## Proposed

```
WHERE CONTAINS(searchVector, @ftsQuery)
```

Search will be powered by:

* MSSQL Full-Text Catalog
* Full-Text Index
* Computed persisted columns for fullName

---

# 6. Technical Design

## 6.1 Database Changes

### 6.1.1 Ensure Full-Text Feature Enabled

```sql
SELECT FULLTEXTSERVICEPROPERTY('IsFullTextInstalled');
```

If not enabled, install SQL Server Full-Text Search feature.

---

### 6.1.2 Create Computed Persisted Columns

### Users Table

Add:

```sql
ALTER TABLE users
ADD fullName AS LTRIM(RTRIM(CONCAT(firstName, ' ', lastName))) PERSISTED;

ALTER TABLE users
ADD fullNameRev AS LTRIM(RTRIM(CONCAT(lastName, ' ', firstName))) PERSISTED;
```

These allow:

* Normal order search
* Reverse order search

---

### 6.1.3 Create Full-Text Catalog

```sql
CREATE FULLTEXT CATALOG ftCatalog AS DEFAULT;
```

---

### 6.1.4 Create Full-Text Indexes

### On Kits

```sql
CREATE FULLTEXT INDEX ON kits(
    kitNumber LANGUAGE 1033,
    name LANGUAGE 1033
)
KEY INDEX PK_kits;
```

### On Family Kits

```sql
CREATE FULLTEXT INDEX ON family_kit(
    kitNumber LANGUAGE 1033,
    name LANGUAGE 1033
)
KEY INDEX PK_family_kit;
```

### On Users

```sql
CREATE FULLTEXT INDEX ON users(
    firstName LANGUAGE 1033,
    lastName LANGUAGE 1033,
    fullName LANGUAGE 1033,
    fullNameRev LANGUAGE 1033
)
KEY INDEX PK_users;
```

Language 1033 = English (adjust if required).

---

# 6.2 Backend Service Changes

## 6.2.1 Search Normalization

Before querying:

* trim
* collapse whitespace
* enforce max length (e.g. 80)
* reject empty string

Example normalization:

```
"  John   Doe " -> "John Doe"
```

---

## 6.2.2 Tokenization Strategy

Split input by whitespace and build a safe FTS query:

```
input: "john doe"
fts:   "\"john*\" AND \"doe*\""
```

Rules:

* Each token gets a prefix wildcard (`*`)
* Tokens are AND-ed to preserve intent
* Escape quotes and reserved characters

---

## 6.2.3 Query Construction

### Kits (non-family)

```
AND CONTAINS((kits.kitNumber, kits.name), @ftsQuery)
OR CONTAINS((user.firstName, user.lastName, user.fullName, user.fullNameRev), @ftsQuery)
```

### Family Kits

```
AND CONTAINS((familyKit.kitNumber, familyKit.name), @ftsQuery)
OR CONTAINS((user.firstName, user.lastName, user.fullName, user.fullNameRev), @ftsQuery)
```

---

## 6.2.4 Fallback Behavior

If FTS is unavailable (feature not installed), fall back to `LIKE` with a warning log.  
Feature-flag this behavior to avoid silent downgrade.

---

# 7. Implementation Plan

### Step 1: Add persisted columns and FTS indexes

* Migration for `users` fullName + fullNameRev
* Full-text catalog + index creation

### Step 2: Add search normalization utility

* Reusable helper in `src/common/utils/search.ts`

### Step 3: Update `GetAllKitService`

* Use normalized input
* Swap `LIKE` blocks with `CONTAINS`
* Preserve existing pagination and ordering

### Step 4: Feature-flag rollout

* Flag: `ENABLE_FTS_KIT_SEARCH`
* Default OFF
* Gradual enablement per environment

---

# 8. Test Plan

### Functional

* Exact match: kit number, kit name, first/last name
* Partial match: prefix tokens
* Multi-word full name search (normal + reversed)
* Mixed case / accents
* Empty or whitespace-only query

### Performance

* Compare FTS vs LIKE on 100k+ rows
* Validate P95 query time stability under load

---

# 9. Rollout Notes

* FTS requires SQL Server Full-Text feature installed.
* Apply migrations in staging first.
* Enable feature flag progressively.
* Monitor query time and CPU before full rollout.

---

# 10. Summary

Admin kits search is correct but **non-scalable** under current `LIKE`-based matching.  
Moving to MSSQL Full-Text Search yields **predictable performance**, **index-backed search**, and **enterprise-grade behavior** while preserving the existing API contract.
