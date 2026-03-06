# Admin Kits Search Review

## Scope
- Feature: Admin dashboard kit search (client, practitioner, family tabs)
- UI path: `/admin` (Dashboard)
- Files reviewed:
  - `src/components/admin/content/Dashboard/index.tsx`
  - `src/components/admin/content/Dashboard/Header.tsx`
  - `src/components/admin/content/Dashboard/Table.tsx`
  - `src/components/admin/content/Dashboard/PractitionerTable.tsx`
  - `src/components/admin/content/Dashboard/FamilyTable.tsx`
  - `src/hooks/useKits.tsx`
  - `src/service/requests/kit.request.ts`

## Verdict
- Status: **Partially working**

## What Works
- Search input is wired correctly (`onChange` updates `search` state).
- Search state is passed to `useKits` as `searchQuery`.
- API call is made to:
  - `GET /kits?page={page}&take={take}&searchQuery={searchQuery}&type={type}&practitionerId={practitionerId}`
- UI renders server response and supports paginated results across tabs.

## Issues Affecting "Search Not Working"
- **Page is not reset when search text changes**.
  - If user is on page > 1 and enters a new query, request stays on the same page.
  - This can return empty results even when matches exist on page 1.
- **Page is not reset when tab changes or filters change**.
  - Switching between Client/Practitioner/Family tabs or applying a practitioner filter keeps the old page index.
  - This can return empty results even when data exists on page 1 for the new tab/filter.
- **Search query is not URL-encoded**.
  - `getKits` uses string interpolation instead of `URLSearchParams`.
  - Special characters (`&`, `#`, `?`, `%`) can break or truncate the query.
- **No debounce**.
  - Sends requests on every keystroke (not a correctness bug, but adds load and instability under latency).

## Final Search Health
- For normal text input on page 1: **works**.
- For users on later pages, switching tabs, or using special characters: **can appear broken**.
