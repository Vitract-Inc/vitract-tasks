# 📘 Practitioner Client Report Search — Frontend Correctness & UX Fixes

# 1. Executive Summary

The frontend search flow for Practitioner Client Reports is **partially working** but fails in common scenarios that make search appear broken. The core issues are:

* **Pagination is not reset** when the search query changes.
* **Search query is not URL-encoded**, causing malformed requests for special characters.
* **No debounce**, resulting in excessive requests and perceived instability.

This document defines a focused remediation plan to make search **correct, deterministic, and stable** while preserving the current API contract.

---

# 2. Problem Statement

Current behavior:

* Search updates `search` state and sends `searchQuery` to the backend.
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

1. Reset pagination to **page 1** on any search query change.
2. Guarantee **URL-safe query encoding** for all inputs.
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
* No changes to pagination semantics beyond reset on query changes.

---

# 5. High-Level Fixes

## Current Flow

```
onChange -> setSearch -> fetch(page, searchQuery)
```

## Proposed Flow

```
onChange -> setSearch -> resetPage(1) -> debounce -> fetch(page=1, encodedQuery)
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

### Step 1: Reset Pagination

* Update the search change handler to:
  * set `page` to 1
  * update `search`
  * trigger fetch for page 1

### Step 2: Use URLSearchParams for Request

* Update the request builder in:
  * `src/service/requests/kit.request.ts`
* Ensure `searchQuery` is appended via `URLSearchParams`.

### Step 3: Add Debounce

* Add debounce to the search input effect:
  * `useDebouncedValue(search, 300)` or local `setTimeout` pattern
* Use debounced value for fetches.

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

The search feature is **not fundamentally broken**, but key UX and correctness gaps make it appear unreliable.  
Implementing the three fixes above will deliver **deterministic results**, **safe requests**, and **stable UX** without backend changes.
