# Phase 1 Risk Elimination Review

## What Needs Improvement
- Define the exact kit identifier source of truth and how it is propagated across routes, stores, and queries to avoid mismatched IDs.
- Specify a migration path for existing persisted Zustand state and React Query cache to prevent stale data after the refactor.
- Clarify whether kit-scoped data should be fully cleared on switch or preserved in-memory for back/forward navigation; current guidance is ambiguous.
- Add explicit error-handling behavior for missing or invalid URL params to avoid undefined behavior on direct links.
- Define concurrency behavior for rapid kit switches or simultaneous edits (e.g., cancel in-flight queries, debounce navigation).
- Specify how to handle optimistic updates and conflict resolution when React Query becomes the single source of truth.
- Provide a clear backend contract for submitted status enforcement, including error codes and UI handling paths.
- Clarify the required audit/logging behavior for submitted-to-draft transitions and who gets notified.
- Define the lifecycle for query invalidation/removal to avoid over-aggressive cache eviction that breaks offline or back navigation.
- Specify which components are responsible for setting `currentKitId` and how to prevent it from being unset in edge routes.
- Add concrete test cases for refresh/back-button flows that include auth state changes and session expiration.
- Identify how to handle external API usage if it is still required; “migration plan” is too vague for execution.
- Add explicit server-side authorization checks for cross-kit access; client-only isolation is insufficient.
- Define measurable success criteria for “no console errors or warnings” to avoid subjective exit checks.
- Provide guidance on feature flags or staged rollout strategy details beyond a placeholder “gradual rollout plan defined.”
- Specify how to prevent data loss when clearing stores on unmount (e.g., save draft, warn on dirty state).
- Define how to avoid race conditions between cache invalidation and navigation (e.g., ensure refetch completes before render).
- Clarify error recovery behavior when `healthInfoId` is missing or does not belong to the current kit.
- Add explicit ownership of query key composition to avoid inconsistent scoping across hooks.

## What Needs To Be Removed
- The “Risk Reduction Estimate” percentages are speculative and not required to complete the phase.
- “Monitoring in place” and “User acceptance testing complete” are rollout requirements; move to a release phase if this phase is purely engineering fixes.
- “Assign owners” and “estimate effort” are project management steps and should not be part of technical exit criteria.
- The explicit grep command example in Task 1.6 is execution detail; keep tooling guidance in a separate implementation notes doc if needed.

## Explicit Constraint Flags
- “These issues must be fixed before any other work” appears in “Why This Phase Exists.”
- “Do NOT start Phase 2 until all Phase 1 exit criteria are met.” appears in “Next Steps.”
- “Only then proceed to Phase 2” appears in “Next Steps.”
- “Phase 1 is complete when ALL of the following are true” appears in “Exit Criteria.”
