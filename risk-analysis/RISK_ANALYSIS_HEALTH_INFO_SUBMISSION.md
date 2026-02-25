# Risk Analysis: Health Info Submission (All Flows)

**Scope:** Customer, Practitioner, and Client-facing health info submission flows

**Last Reviewed:** 2026-02-25

---

## Reasons Health Info Submission Can Fail

1. **API request failures (submit + status update are required)**
Submission requires two sequential calls: `questionRequest.submitQuestions(...)` and then `kitRequest.updateKitStatus(...)` or `kitRequest.updatePractitionerKitStatus(...)`. If either call fails (network error, 4xx/5xx), the UI shows a toast and stops. Partial success is possible (questions saved but status update fails), leaving the kit in an inconsistent state. Refs: `src/components/customer/content/Overview/HealthInfo.tsx`, `src/components/customer/content/EditHealthInfo/HealthInfo.tsx`, `src/components/practitioner/content/RegisterClientKit/HealthInfo.tsx`, `src/components/landing/content/ClientHealthInformation/HealthInfo.tsx`, `src/service/requests/question.request.ts`, `src/service/requests/kit.request.ts`.

2. **Empty or wrong payload due to local store issues**
Submission payload is built from local stores. If the store is empty or holds responses for a different kit, the payload can be empty or mismatched. Customer flows do not filter by kit ID when building the payload, so responses from other kits can leak into the payload. Practitioner flow filters by kit ID, but if SSE data never loads, the payload is empty. Refs: `src/components/customer/content/Overview/HealthInfo.tsx`, `src/components/customer/content/EditHealthInfo/HealthInfo.tsx`, `src/components/landing/content/ClientHealthInformation/HealthInfo.tsx`, `src/components/practitioner/content/RegisterClientKit/HealthInfo.tsx`, `src/store/response.tsx`, `src/store/response-store.ts`.

3. **Submit button disabled because completion state never flips**
`setDisabledStatus(...)` blocks submission if any category is not marked completed. Completion only flips to `true` when the user reaches the end of each category flow. Skipping out early or navigation glitches can leave categories incomplete and permanently disable submission. Refs: `src/components/customer/functions/index.ts`, `src/components/customer/content/Question/GeneralQuestions.tsx`, `src/components/customer/content/Question/MultiSelectQuestions.tsx`, `src/components/customer/content/Question/RadioQuestion.tsx`.

4. **SSE initialization failures (practitioner flow)**
Practitioner health info relies on SSE to load and sync responses. If `joinKit`, `refreshData`, or the SSE connection fails, responses stay empty and submission remains disabled. Refs: `src/components/practitioner/content/RegisterClientKit/HealthInfo.tsx`, `src/store/response-store.ts`, `src/service/commands/health-info.commands.ts`.

5. **SessionStorage unavailable or cleared (customer/client flows)**
Customer and client flows persist responses in `sessionStorage`. If storage is cleared, blocked (private mode), or exceeds limits, responses disappear and submission is blocked or sends an empty payload. Refs: `src/store/response.tsx`, `src/components/customer/content/Overview/HealthInfo.tsx`, `src/components/landing/content/ClientHealthInformation/HealthInfo.tsx`.

6. **Missing kit/user data at submit time**
Submission assumes `user.kit.kitNumber` or `user.practitionerKit.kitNumber` exists. If the kit data is missing or not yet loaded, payload building and status updates can throw or submit with `undefined` IDs. Refs: `src/components/customer/content/Overview/HealthInfo.tsx`, `src/components/practitioner/content/RegisterClientKit/HealthInfo.tsx`.

7. **Client health info prerequisites not met (client-facing link flow)**
Client-facing submission blocks if Terms/Privacy are not accepted or if the sample collection date is missing. Refs: `src/components/landing/content/ClientHealthInformation/HealthInfo.tsx`.

---

## Quick Notes

“Skip all for now” still calls `submitQuestions(...)`. If the payload is empty or malformed, skip can fail the same way as submit. No automatic retry or reconciliation is implemented after failures.
