# Phase 1 Risk Elimination Review

## What Needs Improvement

- Clarify scope boundaries: explicitly list what Phase 1 will not do (e.g., no SSE/WS refactors, no API versioning) to prevent scope creep.
- Add explicit data model ownership: define who writes/reads `order_id`, `user_id`, and `kit_id`, and how they are validated (source system, trust boundary).
- Specify migration strategy for existing Redis-only data (if any) and how to backfill or accept loss; current plan assumes no migration.
- Confirm and enforce “no edits after submission” across all entry points (REST/SSE/WS) and explicitly test it; currently specified but should be flagged as a cross-cutting risk.
- Add concrete error contract for new failures (409 conflict, 400 submitted, 500 DB failure) with example response shape to keep REST/SSE/WS consistent.
- Tighten operational readiness: define minimum dashboard panels, alert routing, and on-call ownership; current monitoring section is broad but not actionable.
- Performance criteria need baseline: include current p95 latency and acceptable delta, not just absolute <200ms target.
- Test plan needs environment assumptions: specify Redis restart test method, DB failover simulation, and required test data volume.
- Security/compliance: add data retention and access control decisions to Task 1.2/1.6 so audit log has clear retention and access policies.
- Feature flag plan: document flag evaluation point and failure mode if DB is unavailable (read/write behavior).
- Document rollback data integrity considerations: if write-through is disabled after partial DB writes, confirm cache still aligns with DB for reads.
- Add explicit transaction isolation expectations to prevent lost updates beyond optimistic locking (e.g., repeatable read for read-modify-write paths).
- Add idempotency for write endpoints or document why not in Phase 1; without it, retries can create duplicate audit entries or overwrite answers unpredictably.
- Define cache consistency rules: whether reads prefer DB when cache is missing/stale and how to reconcile cache vs DB mismatches after failures.
- Validate and sanitize `answer_value` JSON: specify schema/versioning and size limits to avoid malformed payloads and oversized writes.
- Call out PII handling in audit logs: ensure `changes` does not store full answers or sensitive data, or explicitly list redactions.
- Tighten admin override: clarify authorization and audit requirements for `submitted=false` to avoid silent data rollback.
- Clarify time semantics: use UTC consistently for `submitted_at`, `created_at`, `updated_at`, and include clock-skew handling in tests.

## What Needs To Be Removed

- Remove or move non-Phase-1 items from “Exit Criteria” that are not strictly required for risk elimination:
  - “Load test shows <200ms p95 write latency under normal load” (performance tuning is not required to eliminate data loss/corruption).
  - “Zero data loss in 24-hour soak test” (good validation, but not a Phase 1 exit blocker unless defined as compliance requirement).
  - “HIPAA compliance checklist reviewed (if applicable)” (compliance review is important but not strictly part of Phase 1 engineering execution).
  - “Runbook created for on-call engineers” (good operational practice but better placed in Phase 2/3 unless required for go-live).
- Remove the “Deployment can be immediate” wording in Task 1.1/1.5 and replace with standard rollout steps; “immediate” conflicts with the broader phased rollout strategy.
- Remove exact timeline promises (5–6 days) from Phase 1 document or mark them as estimates aligned to resourcing assumptions; current wording implies commitment.

## Explicit “No Edit After Submission” Flags

- Issue list explicitly calls out missing validation: “No validation that submitted=true prevents edits” (H3).
- Task 1.5 explicitly defines the rule: “Reject all write operations when submitted=true to prevent modification of submitted questionnaires.”
- Task 1.5 acceptance criteria explicitly enforces it: “Write operations return 400 Bad Request when submitted=true.”
