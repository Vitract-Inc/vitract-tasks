# Practitioner Search: Observations and Actions Taken

Date: 2026-02-13  
Endpoints under review:
- `GET /practitioners` (service: `GetAllPractitionerProfileService.execute`)
- `GET /kits/practitioner/:practitionerId` (service: `GetAllPractitionerKitService.execute`)

Backend files:
- `src/practitioner/service/get-practitioners.ts`
- `src/kit/service/get-kits-with-practitioner.ts`

## What I observed in the backend implementation

### 1) Route and response contract

`GetAllPractitionerProfileService` returns:

- `practitioners` (array)
- `pageMetaDto` (object)

`GetAllPractitionerKitService` returns:

- `result` (array)
- `pageMetaDto` (object)

`pageMetaDto` includes:

- `page`
- `take`
- `itemCount`
- `pageCount`
- `hasPreviousPage`
- `hasNextPage`

### 2) Search behavior implemented

Search runs only if `searchQuery` is truthy.

Practitioner search (`get-practitioners.ts`):

- Pattern used: `%${searchQuery}%`
- Search fields:
  - `practitioner.practiceName`
  - `user.firstName`
  - `user.lastName`

Practitioner kits search (`get-kits-with-practitioner.ts`):

- Pattern used: `%${searchQuery}%`
- Search fields:
  - `kits.kitNumber`
  - `kits.name`
  - `practitionerUser.firstName`
  - `practitionerUser.lastName`

There is **no input normalization** (trim, whitespace collapse, max length) and no explicit collation for case/accent handling.

### 3) Scope and filtering

Practitioner search:

- Filters only `user.status = ACTIVE`
- Ordered by `practitioner.createdAt DESC`

Practitioner kits search:

- Validates `practitionerId` exists
- Filters by:
  - `kits.practitionerId = :practitionerId`
  - `kits.ownerType = PRACTITIONER`
- Ordered by `kits.createdAt DESC`

### 4) Pagination and determinism

- Pagination uses `take` and `skip` from `PageOptionsDto`.
- Ordering is deterministic.
- Results and count are derived from the same query (`getManyAndCount()`).

This means if the UI requests a page out of range for the filtered dataset, the backend will return an empty array with a non-zero `itemCount`.

### 5) Input safety

Search uses bound parameters (`:searchQuery`), which is safe from SQL injection.  
URL encoding of special characters remains a **client responsibility**.

## What I did

1. Reviewed both service files to confirm search, pagination, and filters.
2. Identified scaling and correctness risks for `%LIKE%` search across multiple columns.
3. Documented a full-text search upgrade path (see `practitioner_backend_fix.md`).

## Key clarification for UI behavior

If the frontend does not reset to page 1 when search changes, it can legitimately receive:

- `result: []`
- `itemCount > 0`

This is expected pagination behavior and should not be interpreted as a backend search failure.
