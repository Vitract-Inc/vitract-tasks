# Order Test Contract Mismatch Audit (`isSubPractitioner`)

## Audience
Engineering Manager (EM), Frontend, Backend, QA.

## Date
March 9, 2026.

## Executive Summary
Practitioner order submission failed in the Pay-As-You-Go flow because frontend sent `isSubPractitioner` in `POST /orders`, and backend validation rejected it with:
- `property isSubPractitioner should not exist`

This indicates DTO/contract drift between frontend request shape and backend accepted fields for order creation.

## Incident Signal
Observed on route:
- `/practitioner/orders/form/pay-as-you-go`

User-facing error:
- `property isSubPractitioner should not exist`

## Scope
- Confirmed affected flow: Practitioner Order Test, Pay-As-You-Go form submit.
- Likely additional risk: Monthly Billing and Kits-On-Site forms, since they share the same optional sub-practitioner fields and submit through the same endpoint family.

Relevant frontend paths:
- `src/components/practitioner/content/Orders/OrderTest/Forms/PayAsYouGoForm.tsx`
- `src/components/practitioner/content/Orders/OrderTest/Forms/MonthlyBillingForm.tsx`
- `src/components/practitioner/content/Orders/OrderTest/Forms/KitsOnSiteForm.tsx`
- `src/service/requests/order.request.ts`
- `src/service/types/order.interface.ts`

## Contract Matrix (As Observed)
| Endpoint | Frontend sends `isSubPractitioner` | Backend behavior |
|---|---:|---|
| `POST /orders` | Yes (from form values spread) | Rejects (`should not exist`) |

## Reproduction (Pay-As-You-Go)
1. Navigate to Practitioner Order Test Pay-As-You-Go form.
2. Enter valid order/shipping details.
3. Submit.
4. Frontend payload includes `isSubPractitioner` (and may include `subPractitionerName`).
5. Backend rejects with validation message: `property isSubPractitioner should not exist`.

## Root Cause Analysis
In Pay-As-You-Go submit logic, payload is built via object spread from form values (`...vals`) before endpoint call. Form values include:
- `isSubPractitioner` (boolean)
- `subPractitionerName` (string)

`orderRequest.orderTest(data)` posts this shape directly to `POST /orders`.

If backend DTO no longer permits `isSubPractitioner` (or only permits it in specific contexts), strict validation rejects the request.

Root cause is **frontend payload over-sharing** + **backend DTO contract drift**.

## Impact
- Practitioner unable to complete Pay-As-You-Go order from UI.
- Revenue-impacting funnel interruption for self-serve order flow.
- Increased support contact and manual ordering overhead.

## Risk Assessment
- Regression risk: Medium to High if all order form variants continue sharing broad payload shapes.
- Silent-failure risk: Medium if backend later strips unknown fields silently, creating business-logic ambiguity around sub-practitioner attribution.

## Decision Options
1. Backend schema alignment (long-term):
   - Explicitly define whether `isSubPractitioner` and `subPractitionerName` are supported on `POST /orders` and for which order/delivery modes.
2. Frontend endpoint-scoped payload shaping (short-term):
   - Whitelist allowed fields before submit; do not spread full form object.
3. Hybrid (recommended):
   - Immediate frontend payload hardening + backend API spec harmonization.

## Recommendation
Use the hybrid path:
- Short term: frontend should construct explicit DTO per endpoint/order type and exclude unsupported fields.
- Long term: backend should publish/maintain canonical API schema and enforce consistency across order modes.

## Required Follow-ups
- Backend:
  - Confirm authoritative `POST /orders` create DTO.
  - State clear policy for `isSubPractitioner`/`subPractitionerName` (required/optional/forbidden and by mode).
- Frontend:
  - Remove spread-based submit payloads in order forms.
  - Add request-body regression tests to verify allowed fields only.
- QA:
  - Add submit-path checks for Pay-As-You-Go, Monthly Billing, and Kits-On-Site.
- Product/EM:
  - Assign contract ownership and sign-off gate for FE/BE schema changes affecting order creation.

## Rollback / Mitigation
If fix rollout introduces regressions:
- Roll back frontend payload-shaping change.
- Apply backend temporary tolerance for extra fields while final contract is aligned.
- Keep support runbook updated for interim user guidance.

## Appendix A: Example Payloads
### Payload currently sent by frontend (representative)
```json
{
  "firstName": "Venice",
  "lastName": "Johnson",
  "email": "",
  "addressLineOne": "458 Rock Hill Road",
  "addressLineTwo": "",
  "country": "United States",
  "state": "Missouri",
  "city": "Jefferson City",
  "postalCode": "65109",
  "quantity": 1,
  "isSubPractitioner": false,
  "subPractitionerName": "",
  "kitType": "...",
  "paymentAction": "PaymentLink",
  "orderType": "pay-as-you-go",
  "currency": "USD",
  "deliveryMode": "dropship"
}
```

### Payload expected by stricter backend DTO (example)
```json
{
  "firstName": "Venice",
  "lastName": "Johnson",
  "addressLineOne": "458 Rock Hill Road",
  "addressLineTwo": "",
  "country": "United States",
  "state": "Missouri",
  "city": "Jefferson City",
  "postalCode": "65109",
  "quantity": 1,
  "kitType": "...",
  "paymentAction": "PaymentLink",
  "orderType": "pay-as-you-go",
  "currency": "USD",
  "deliveryMode": "dropship"
}
```

## Appendix B: EM Summary (One-Liner)
Pay-As-You-Go practitioner order creation failed on March 9, 2026 because frontend included `isSubPractitioner` in `POST /orders`, but backend validation now rejects that field, exposing order-create DTO contract drift.
