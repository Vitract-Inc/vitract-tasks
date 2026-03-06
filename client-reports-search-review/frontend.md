# Client Report Search Review

## Scope
- Feature: Practitioner Client Report search
- UI path: `/practitioner` and `/practitioner/test-reports`
- Files reviewed:
  - `src/components/practitioner/content/ClientReport/Header.tsx`
  - `src/components/practitioner/content/ClientReport/index.tsx`
  - `src/hooks/usePractitionerKits.tsx`
  - `src/service/requests/kit.request.ts`

## Verdict
- Status: **Partially working**

## What Works
- Search input is wired correctly (`onChange` updates `search` state).
- Search state is sent to backend as `searchQuery`.
- API call is made to:
  - `GET /kits/practitioner-kits?page={page}&take={take}&searchQuery={searchQuery}`
- UI renders server response and supports paginated results.

## Issues Affecting "Search Not Working"
- **Page is not reset when search text changes**.
  - If user is on page > 1 and enters a new query, request stays on the same page.
  - This can return empty results even when matches exist on page 1.
- **Search query is not URL-encoded via `URLSearchParams`**.
  - Special characters (e.g. `&`, `#`, `?`) can produce malformed/ambiguous query strings.
- **No debounce**.
  - Sends request on every keystroke (not a correctness bug, but can amplify backend load and perceived instability under latency).

## Final Search Health
- For normal text input on page 1: **works**.
- For users on later pages or using special characters: **can appear broken**.
