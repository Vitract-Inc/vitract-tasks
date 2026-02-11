# 📘 Practitioner Kit Search — Enterprise Full-Text Search Upgrade (MSSQL)**


# 1. Executive Summary

The current implementation of `GET /kits/practitioner-kits` uses `LIKE '%query%'` across multiple columns with OR conditions. This approach:

* Does not scale
* Prevents effective index usage
* Performs full scans at moderate dataset sizes
* Has unpredictable performance
* Is not enterprise-grade

This PRD defines the migration to **MSSQL Full-Text Search (FTS)** to deliver:

* Scalable, indexed search
* Multi-word token support
* Full name search (`firstName + lastName`)
* Deterministic, secure, predictable behavior
* Gold-standard search architecture

---

# 2. Problem Statement

Current search:

* Uses `%query%` across:

  * `kits.kitNumber`
  * `kits.name`
  * `user.firstName`
  * `user.lastName`
* Applies OR logic
* Applies collation at query time
* Does not scale

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
3. Maintain existing authorization scope logic.
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

---

## 6.1 Database Changes

### 6.1.1 Ensure Full-Text Feature Enabled

```sql
SELECT FULLTEXTSERVICEPROPERTY('IsFullTextInstalled');
```

If not enabled, install SQL Server Full-Text Search feature.

---

## 6.1.2 Create Computed Persisted Columns

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

## 6.1.3 Create Full-Text Catalog

```sql
CREATE FULLTEXT CATALOG ftCatalog AS DEFAULT;
```

---

## 6.1.4 Create Full-Text Indexes

### On Kits

```sql
CREATE FULLTEXT INDEX ON kits(
    kitNumber LANGUAGE 1033,
    name LANGUAGE 1033
)
KEY INDEX PK_kits;
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
"  John   Doe " → "John Doe"
```

---

## 6.2.2 Tokenization Strategy

Split by whitespace:

Input:

```
"john doe"
```

Tokens:

```
["john", "doe"]
```

Construct FTS query:

```
"john*" AND "doe*"
```

Prefix search ensures partial matching.

---

## 6.2.3 Query Builder Change

Replace current LIKE block with:

```sql
CONTAINS(
    (kits.kitNumber, kits.name, users.firstName, users.lastName, users.fullName, users.fullNameRev),
    @ftsQuery
)
```

Authorization scope remains applied before search.

---

## 6.2.4 Preserve Ordering

Keep:

```
ORDER BY kits.createdAt DESC, kits.id DESC
```

FTS ranking is NOT used initially (no relevance sort yet).

---

# 7. Authorization Integrity

Search MUST be applied after practitioner visibility filtering.

Order of logic:

1. Scope by practitioner access
2. Apply FTS predicate
3. Apply ordering
4. Apply pagination

This prevents search from leaking unauthorized rows.

---

# 8. API Contract

No breaking changes.

Optional enhancement (recommended but not required):

Add in meta:

```
normalizedSearchQuery: string | null
```

Optional:

```
isPageOutOfRange: boolean
```

---

# 9. Performance Expectations

### Before

* Scan-based LIKE
* Performance degrades with row growth

### After

* Index-backed FTS
* Stable performance
* O(log n) search complexity (index-based)

Expected:

* Sub-100ms query at 1M+ rows (infra-dependent)

---

# 10. Migration Plan

### Phase 1 – Schema

* Add computed columns
* Create catalog
* Create FTS indexes

### Phase 2 – Backend Refactor

* Implement normalization
* Implement tokenization
* Replace LIKE with CONTAINS

### Phase 3 – Testing

* Integration tests
* Load testing
* Query plan verification

### Phase 4 – Rollout

* Deploy schema migration
* Deploy backend update
* Monitor slow query logs

---

# 11. Test Plan

## Functional Tests

* Exact match
* Partial match
* Full name match
* Reverse full name match
* Single token match
* Mixed case
* Accent variations
* No match

## Security Tests

* Injection attempts
* Long query rejection
* Wildcard characters

## Performance Tests

* 10k rows
* 100k rows
* 1M rows

---

# 12. Observability

Add structured logging:

* practitionerId
* normalizedQuery length
* page
* take
* resultCount
* executionTime

Monitor:

* Query duration
* Deadlocks
* CPU spikes

---

# 13. Risks

| Risk                         | Mitigation                       |
| ---------------------------- | -------------------------------- |
| FTS not enabled in prod      | Infra verification before deploy |
| Migration lock               | Run during low traffic window    |
| Incorrect token construction | Unit test tokenization           |
| Search semantics change      | Document behavior                |

---

# 14. Success Criteria

* No LIKE in practitioner search
* No per-query COLLATE usage
* FTS index present and used in query plan
* Stable performance under load
* All integration tests pass
* No unauthorized data leakage

---

# 15. Final State (Gold Standard Definition)

✔ Column-level collation
✔ Computed persisted fullName columns
✔ MSSQL Full-Text Search
✔ Tokenized AND search
✔ Prefix support
✔ Auth-first filtering
✔ Deterministic ordering
✔ Validated input
✔ Capped pagination
✔ Structured observability
