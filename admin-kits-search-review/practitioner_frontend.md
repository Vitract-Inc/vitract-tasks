# Admin Practitioner Search Review

## Scope
- Feature: Admin practitioner search + practitioner client kits search
- UI paths:
  - `/admin/practitioners` (PractitionerInfo)
  - `/admin/practitioner-clients` (PractitionerClientsReport)
- Files reviewed:
  - `src/components/admin/content/PractitionerInfo/index.tsx`
  - `src/components/admin/content/PractitionerClientsReport/index.tsx`
  - `src/hooks/usePractitionersWithPagination.tsx`
  - `src/hooks/usePractitionerClientsReport.tsx`
  - `src/service/requests/practitioner.request.ts`
  - `src/service/requests/kit.request.ts`

## Verdict
- Status: **Partially working**

## What Works
- Practitioner clients search uses a **debounced** input (300ms) and resets page to 1.
- Practitioner clients requests are **URL-encoded** via `URLSearchParams`.
- Both screens render server responses and support paginated results.

## Issues Affecting "Search Not Working"
- **Practitioner list page does not reset to page 1 when search changes**.
  - If user is on page > 1 and enters a new query, request stays on the same page.
  - This can return empty results even when matches exist on page 1.
- **Practitioner list search query is not URL-encoded**.
  - `fetchPractitionersWithPagination` uses string interpolation instead of `URLSearchParams`.
  - Special characters (`&`, `#`, `?`, `%`) can break or truncate the query.
- **Practitioner list search has no debounce**.
  - Sends requests on every keystroke (not a correctness bug, but adds load and instability under latency).

## Final Search Health
- Practitioner clients report: **works** under normal usage.
- Practitioner list: **can appear broken** when on later pages or using special characters.
