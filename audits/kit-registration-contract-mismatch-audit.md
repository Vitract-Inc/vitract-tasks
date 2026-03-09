# Kit Registration Contract Mismatch Audit (`lockStatus`)

## Audience
Engineering Manager (EM), Frontend, Backend, QA.

## Date
March 9, 2026.

## Executive Summary
A frontend registration payload included `lockStatus` for both kit registration endpoints.

Observed behavior indicates endpoint contract mismatch:
- `POST /kits` accepts (or expects) `lockStatus`.
- `POST /kits/practitioner-kit` rejects `lockStatus` with validation error: `property lockStatus should not exist`.

This caused practitioner registration failures even when form inputs were valid.

## Incident Signal
User-facing toast surfaced backend validation error during Practitioner "Register Client Kit" submit.

Example error:
- `property lockStatus should not exist`

## Scope
- Affected flow: Practitioner client-kit registration UI.
- Potentially unaffected flow: Customer self-kit registration (`POST /kits`) if backend contract still accepts `lockStatus` there.
- Components involved:
  - `src/components/practitioner/content/RegisterClientKit/RegisterKit.tsx`
  - `src/components/customer/content/Overview/RegisterKit.tsx`
  - `src/service/requests/kit.request.ts`
  - `src/service/types/kit.interface.ts`

## Contract Matrix (As Observed)
| Endpoint | Frontend sends `lockStatus` | Backend behavior |
|---|---:|---|
| `POST /kits` | Yes | Accepts / expected |
| `POST /kits/practitioner-kit` | Yes | Rejects (`should not exist`) |

## Reproduction (Practitioner)
1. Open Practitioner "Register Client Kit".
2. Enter valid kit number, client name, and date.
3. Submit.
4. Request body includes `lockStatus`.
5. Backend rejects with validation message: `property lockStatus should not exist`.

## Root Cause Analysis
Frontend shares a common registration payload construction path and currently forwards object fields directly into the request body. Both registration forms add `lockStatus` to their submit payload object.

Because backend DTO rules differ between the two endpoints, identical frontend payload shape is incompatible:
- Compatible with `/kits`
- Incompatible with `/kits/practitioner-kit`

Root cause is therefore **cross-endpoint payload contract drift** + **frontend over-sharing fields**.

## Impact
- Practitioner registration blocked for affected users.
- Increased support burden and failed task completion.
- Risk of repeated regressions while endpoint schemas remain unsynchronized.

## Risk Assessment
- Regression risk: Medium if frontend continues to reuse one broad payload shape for endpoints with different DTOs.
- Silent-failure risk: Low to Medium. Backend currently fails loudly; however, future partial acceptance could hide state inconsistencies.

## Decision Options
1. Backend contract alignment (preferred long-term):
   - Make both endpoints accept the same registration schema, including explicit policy for `lockStatus`.
2. Frontend endpoint-specific payload shaping (preferred short-term):
   - Send only fields allowed by each endpoint.
3. Hybrid:
   - Immediate frontend shaping + scheduled backend schema harmonization.

## Recommendation
Proceed with the hybrid approach:
- Short term: frontend sends endpoint-specific payloads (or whitelists fields per endpoint).
- Long term: backend publishes a single source-of-truth contract for registration DTOs and aligns validation behavior across endpoints.

## Required Follow-ups
- Backend:
  - Confirm authoritative schema for each endpoint.
  - Clarify whether `lockStatus` is required, optional, or forbidden per route.
- Frontend:
  - Enforce request-field whitelisting for registration endpoints.
  - Add regression tests that assert request bodies per endpoint.
- QA:
  - Add test coverage for both customer and practitioner registration flows with contract assertions.
- Product/EM:
  - Approve contract ownership model to prevent future drift (API spec first, FE generated/validated types where possible).

## Rollback / Mitigation
If implementation rollout causes unexpected behavior:
- Roll back to prior frontend request mapping.
- Keep temporary user guidance in support runbook.
- Prioritize backend hotfix to tolerate extra fields until frontend update is redeployed.

## Appendix A: Example Payloads
### Payload currently sent by frontend
```json
{
  "kitNumber": "00368302",
  "name": "Karli Cessario",
  "dateOfSampleCollection": "2026-03-09T00:00:00.000Z",
  "lockStatus": "locked"
}
```

### Payload expected by stricter practitioner endpoint (example)
```json
{
  "kitNumber": "00368302",
  "name": "Karli Cessario",
  "dateOfSampleCollection": "2026-03-09T00:00:00.000Z"
}
```

## Appendix B: EM Summary (One-Liner)
Practitioner kit registration failed on March 9, 2026 because frontend sent `lockStatus` to `POST /kits/practitioner-kit`, but that endpoint rejects the field while `POST /kits` accepts it, exposing endpoint DTO contract drift.
