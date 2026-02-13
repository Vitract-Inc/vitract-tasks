# 📘 Admin Practitioner Search — Frontend Correctness & UX Fixes

# 1. Executive Summary

The admin practitioner search flow is **partially working** but fails in common scenarios that make search appear broken. The core issues are:

* **Pagination is not reset** when the search query changes (Practitioner list).
* **Search query is not URL-encoded** for practitioner list requests.
* **No debounce** on practitioner list search, causing request floods.

This document defines a focused remediation plan to make search **correct, deterministic, and stable** while preserving the current API contract.

---

# 2. Problem Statement

Current behavior:

* Practitioner list search updates `search` state and sends `searchQuery` to the backend.
* Page state is **not reset**, so filtered results often query an out-of-range page.
* `searchQuery` is appended without URL encoding, breaking queries like:
  * `john & jane`
  * `#123`
  * `ACME?`
* Each keystroke triggers a request with no debounce.

Result:

* Users see empty results and assume search is broken.
* Backend receives malformed query strings for certain inputs.
* Unstable UX under latency or backend load.

---

# 3. Goals

### Functional Goals

1. Reset pagination to **page 1** on search query change (practitioner list).
2. Guarantee **URL-safe query encoding** for practitioner list requests.
3. Add **debounce** to reduce request volume without sacrificing responsiveness.
4. Preserve existing API contract and response handling.

### Non-Functional Goals

1. Predictable UX for fast typing and slower connections.
2. No backend changes required.
3. Minimal regression risk.

---

# 4. Non-Goals

* No changes to backend filtering logic.
* No new search ranking or fuzzy matching.
* No change to pagination semantics beyond reset on search changes.

---

# 5. High-Level Fixes

## Current Flow (Practitioner List)

```
onChange -> setSearch -> fetch(page, searchQuery)
```

## Proposed Flow

```
onChange -> setSearchInput -> debounce -> setSearch + resetPage(1) -> fetch(page=1, encodedQuery)
```

---

# 6. Technical Design

## 6.1 Reset Page on Search Change

When the search input changes:

* Set `page = 1`
* Trigger fetch with the new query

This ensures filtered results start from a valid page.

---

## 6.2 URL-Encode Query String

Ensure all query values are passed via `URLSearchParams` or `encodeURIComponent`.

Example:

```
searchQuery="john & jane" -> "john%20%26%20jane"
```

---

## 6.3 Debounce Search Requests

Add a small debounce (e.g. 250–400ms) before issuing requests to:

* Reduce request flood on fast typing
* Avoid race conditions / flicker
* Improve perceived stability

---

# 7. Implementation Plan

### Step 1: Add Debounced Search State

* Update `src/components/admin/content/PractitionerInfo/index.tsx`:
  * Add `searchInput` + `search` split state (same pattern as `PractitionerClientsReport`)
  * Debounce with `setTimeout(300)`
  * Reset `currentPage` to 1 on query change

### Step 2: Use URLSearchParams for Practitioner Requests

* Update `src/service/requests/practitioner.request.ts`:
  * Build query string with `URLSearchParams` in `fetchPractitionersWithPagination`

### Step 3: Use URLSearchParams for Revalidator Key (Optional)

* Update `src/hooks/usePractitionersWithPagination.tsx`:
  * Build the revalidator key with `URLSearchParams` to match encoded query behavior

---

# 8. Test Plan

### Manual

1. Search on page 3 with a narrow query:
   * Expected: results load from page 1
2. Query with special characters:
   * `john & jane`, `#123`, `ACME?`
   * Expected: backend receives a valid request
3. Fast typing under latency:
   * Expected: no request flood, stable UI

### Automated (if existing patterns allow)

* Unit test for search change resets page to 1
* Request builder test for encoded query string

---

# 9. Rollout Notes

* Safe to deploy independently of backend changes
* No API contract changes
* Low regression risk if pagination and request handling are covered by tests

---

# 10. Summary

The admin practitioner search is **not fundamentally broken**, but key UX and correctness gaps make it appear unreliable.  
Implementing the fixes above will deliver **deterministic results**, **safe requests**, and **stable UX** without backend changes.
