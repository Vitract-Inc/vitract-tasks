# Admin Kits Search: Observations and Actions Taken

Date: 2026-02-13  
Endpoint under review: `GET /kits` (service: `GetAllKitService.execute`)  
Backend file: `src/kit/service/get-kits.ts`

## What I observed in the backend implementation

### 1) Route and response contract

The service returns:

- `result` (array)
- `pageMetaDto` (object)

`pageMetaDto` includes:

- `page`
- `take`
- `itemCount`
- `pageCount`
- `hasPreviousPage`
- `hasNextPage`

This aligns with existing pagination contract.

### 2) Search behavior implemented

Search runs only if `searchQuery` is truthy:

- Pattern used: `%${searchQuery}%`
- Search fields:
  - `kits.kitNumber`
  - `kits.name`
  - `user.firstName`
  - `user.lastName`
  - `familyKit.kitNumber`
  - `familyKit.name`
  - `user.firstName`
  - `user.lastName`

There is **no input normalization** (trim, whitespace collapse, max length) and no explicit collation for case/accent handling.

### 3) Type filtering and scope

The query is split by `type`:

- `FAMILY`: uses `familyKitRepo` and orders by `familyKit.dateOfSampleCollection`
- `PRACTITIONER`: filters by `ownerType = PRACTITIONER` and optional `practitionerId`
- `CLIENTS`: filters by `ownerType = CLIENT`

Search is applied after the type filtering.

### 4) Pagination and determinism

- Pagination uses `take` and `skip` from `PageOptionsDto`.
- Ordering is deterministic:
  - `dateOfSampleCollection DESC`
- Results and count are derived from the same query builder (`getMany` + `getCount`).

This means if the UI requests a page that is out of range for the filtered dataset, the backend will correctly return an empty `result` with a non-zero `itemCount`.

### 5) Input safety

Search uses bound parameters (`:searchQuery`), which is safe from direct SQL injection.
However, URL encoding of special characters is a **frontend responsibility**.

## What I did

1. Reviewed `src/kit/service/get-kits.ts` to confirm search and pagination behavior.
2. Identified scaling and correctness risks for `%LIKE%` search across multiple columns.
3. Documented upgrade path using full-text search (see `backend_fix.md`).

## Key clarification for UI behavior

If the frontend does not reset to page 1 when search changes, it can legitimately receive:

- `result: []`
- `itemCount > 0`

This is expected pagination behavior and should not be interpreted as a backend search failure.
