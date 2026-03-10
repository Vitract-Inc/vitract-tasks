# E2E Proof Report: `Practitioner Tab Undefined` on `/admin/orders`

## Date
March 10, 2026.

## Objective
Prove, with E2E evidence, when the `undefined undefined` symptom appears in **Admin → Orders → Practitioners** and validate that it is tied to a single rendering condition.

## Scope
- Surface: `/admin/orders` → `practitioners` tab
- Column under test: `Practitioner's full name`
- Test file: `e2e/admin-orders-practitioner-undefined.spec.ts`

## Test Strategy
A deterministic Playwright E2E test was added with API route mocks for:
- `GET /users/profile` (admin auth/permissions)
- `GET /orders` (practitioner orders list)
- `GET /orders/customer-orders` (non-blocking fallback for default tab load)

The test drives three server payload scenarios via `searchQuery`:
1. **Both missing**: `subPractitionerName = ""`, `user.firstName/lastName` absent
2. **Sub-practitioner present**: `subPractitionerName = "Dr Sub Name"`, user names absent
3. **User names present**: `subPractitionerName = ""`, `user.firstName/lastName = Grace Hopper`

## Command Run
```bash
pnpm exec playwright test e2e/admin-orders-practitioner-undefined.spec.ts --project=chromium
```

## Result
```text
1 passed (chromium)
```

## Evidence
Observed assertions from E2E run:
- Scenario 1 (both missing): UI renders `undefined undefined` exactly once.
- Scenario 2 (sub-practitioner present): UI shows `Dr Sub Name`; `undefined undefined` count is `0`.
- Scenario 3 (user names present): UI shows `Grace Hopper`; `undefined undefined` count is `0`.

This isolates the symptom to the condition where both fallback sources are absent in the practitioner-name display expression.

## Source Mapping
The tested UI behavior corresponds to this expression:
- `content.subPractitionerName || \`${content?.user?.firstName} ${content?.user?.lastName}\` || "N/A"`

Location:
- `src/components/admin/content/Orders/OrderHistory/PractitionerOrders/Table/PractitionerTable.tsx`

Equivalent pattern also exists in:
- `src/components/admin/content/Orders/OrderHistory/PractitionerOrders/Table/PractitionerMonthlyBillingTable.tsx`
- `src/components/admin/content/Orders/OrderHistory/PractitionerOrders/Table/PractitionerKitsOnSiteTable.tsx`

## Conclusion
Within the practitioner tab name-rendering path validated by this E2E, `undefined undefined` occurs only when:
- `subPractitionerName` is falsy, and
- `user.firstName` and `user.lastName` are absent.

No `undefined undefined` appears when either fallback source is provided.

## Notes / Limits
- This proof is scoped to the practitioner name rendering in the practitioner orders tab.
- It does not claim that no other unrelated component in the application could render `undefined` text from different logic.
