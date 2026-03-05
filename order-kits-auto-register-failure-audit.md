# Order Kits Auto-Register Failure Audit

## Executive Summary
- Base: `{BASE_BRANCH}` **[NEEDS CONTEXT: PR base branch not provided]**
- Target: `staging`
- PRD: `Not provided`
- P0 Issues: `2`
- Overall Production Risk: `High`
- Data/Query Scale Risk: `Medium`
- Concurrency Risk: `High`

Auto-register can fail or behave incorrectly in the shipment update path because enqueue gating is too broad and kit list handling is not null-safe. The current logic can enqueue auto-register for non-dropship modes and can crash when `kitIds` is omitted on partial updates. Under concurrent shipment updates, duplicate jobs can still be scheduled because enqueue is not idempotent by `(orderId, kitId)`.

---

## Must-Fix (P0)
> Only items that must block merging.

### src/order/service/update-order.ts
- **[P0][BUG] ~160**
  - **Issue:** Auto-register gating uses `OrderRecord.deliveryMode !== DeliveryMode.ON_SITE` instead of explicit dropship check.
  - **Production failure scenario:** Any non-`ON_SITE` value (future enum, malformed legacy value, null-like migration residue) will incorrectly enqueue auto-registration, causing registrations for unsupported fulfillment modes.
  - **Fix:** Restrict enqueue condition to `OrderRecord.deliveryMode === DeliveryMode.DROPSHIP`.
  - **Fix snippet (if required):**
    ```ts
    if (OrderRecord.deliveryMode === DeliveryMode.DROPSHIP) {
      await this.queueService.addAutoRegisterPractitionerOrderKitsJobsBulk(jobs);
    }
    ```

### src/order/service/update-order.ts
- **[P0][BUG] ~63-71, ~161**
  - **Issue:** `data.kitIds` is optional in `UpdateOrderDto`, but code assumes it is always defined (`new Set(data.kitIds)`, `data.kitIds.map(...)`).
  - **Production failure scenario:** Shipment-only updates (tracking fields without `kitIds`) throw at runtime, rollback transaction, and skip both email/queue side effects; affected orders never auto-register.
  - **Fix:** Default to persisted kit list when `kitIds` is absent, and never map/filter on nullable input.
  - **Fix snippet (if required):**
    ```ts
    const requestedKitIds = data.kitIds ?? OrderRecord.orderKits.map((k) => k.kitId);
    const incomingKitIds = new Set(requestedKitIds);
    const kitsToAdd = requestedKitIds.filter((k) => !existingKitIds.has(k));
    // ...
    const jobs = requestedKitIds.map((kitId) => ({ orderId: OrderRecord.id, kitId }));
    ```

---

## Should-Fix (P1)

### src/queues/services/queue.service.ts
- **[P1][RACE] ~215-274**
  - **Issue:** Auto-register enqueue methods do not set deterministic `jobId` (e.g., `auto-register:{orderId}:{kitId}`), so duplicate submissions/retries create duplicate jobs.
  - **Failure scenario:** Concurrent updates that both observe `trackingEmailStatus=false` enqueue duplicate jobs; at scale this amplifies queue load, repeated DB writes, and dead-letter noise.
  - **Fix:** Enforce idempotent job IDs in both single and bulk enqueue paths.
  - **Fix snippet (if required):**
    ```ts
    const jobId = `auto-register:${d.orderId}:${d.kitId}`;
    opts: { jobId, priority: 8, attempts: 3, backoff: { type: 'exponential', delay: 2000 }, ...options }
    ```

### src/order/service/update-order.ts
- **[P1][RACE] ~41-47, ~117-127, ~160-168**
  - **Issue:** `justShipped` is computed from an unlocked read; concurrent transactions can both pass the first-shipment gate and enqueue jobs.
  - **Failure scenario:** Two admins update tracking around the same time, both transactions enqueue registration jobs for the same kits.
  - **Fix:** Use a conditional atomic update (`WHERE id=:id AND trackingEmailStatus=0`) and only enqueue when affected row count is `1`; alternatively lock the row (`pessimistic_write`) before evaluating shipment transition.

---

## Nice-to-Have (P2)

### src/order/service/update-order.ts
- **[P2][QUALITY] ~42-45**
  - **Concern:** `relations: ['orderKits', 'user']` loads full relation payload even though this path needs a narrow subset.
  - **Impact at scale:** Extra memory/serialization overhead on a hot mutation endpoint under high throughput.
  - **Improvement:** Select only required columns for `order`, `orderKits.kitId`, and `user` fields used by recipient logic.

---

## Category Details (No Duplication)

### 1) Race Conditions
- `src/queues/services/queue.service.ts ~215-274` (non-idempotent job enqueue)
- `src/order/service/update-order.ts ~41-47, ~117-127, ~160-168` (first-shipment gate race)

### 2) Bugs
- `src/order/service/update-order.ts ~160` (delivery mode predicate too broad)
- `src/order/service/update-order.ts ~63-71, ~161` (`kitIds` optionality mismatch -> runtime failure)

### 3) Bad Practices
- No blocking bad practices identified.

### 4) Industry-Level Quality
- `src/order/service/update-order.ts ~42-45` (over-fetch on mutation path)

---

## Index & Query Optimization Notes (Required When DB/ORM/SQL Is Touched)

### Findings
- `src/order/service/update-order.ts:42-45` — lookup is `findOne` by `id` (PK, index-friendly), but relation eager-load width is larger than needed for this hot path.
- `src/order/service/update-order.ts:84-87` — delete uses `orderId` + `kitId IN (...)`; current `order-kits` has index only on `orderId`, so high-cardinality deletes/filtering on both keys can degrade.

### Index Recommendations (If Needed)
- Proposed index: `order-kits(orderId, kitId)` (unique if business allows one kit per order)
- Justification: supports `DELETE ... WHERE orderId = ? AND kitId IN (...)` and future exact-match lookups.
- Migration note: create index first, then rely on it in production hot paths; if making unique, deduplicate existing rows before adding constraint.

---

## PRD Alignment (Only if PRD Provided)

### Confirmed
- Not applicable.

### Deviations
- Not applicable.

---

## Required Risk-Based Test Plan
Only tests justified by findings:
- Unit:
  - `update-order` should not throw when `kitIds` is omitted and tracking fields are present.
  - auto-register enqueue should run only when `deliveryMode === DROPSHIP`.
  - queue service should generate deterministic `jobId` for single and bulk paths.
- Integration:
  - update order with dropship + multiple kits enqueues one job per kit.
  - update order with on-site does not enqueue auto-register jobs.
- Concurrency (if applicable):
  - two simultaneous shipment updates for same order result in one effective enqueue set.
- Query regression (EXPLAIN/ANALYZE) (if applicable):
  - validate `order-kits(orderId, kitId)` is used for delete/filter path.
- Load/stress (if applicable):
  - bulk enqueue 1k+ kits across concurrent orders confirms no duplicate-job explosion.
- Migration/rollout safety (if applicable):
  - if adding unique `(orderId, kitId)`, run pre-migration duplicate detection and cleanup.

---

## Final Verdict
⚠️ Merge after P0 fixes

Current shipment update logic can both mis-gate auto-registration and crash on valid partial updates, which is unsafe for production fulfillment workflows.
