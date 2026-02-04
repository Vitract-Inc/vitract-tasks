# Code Review: Duplicate PaymentStatements Risk (Monthly Billing)

## 1) Summary conclusion
**Duplicates are possible** under real execution paths, even with the current locking and unique index. Specifically:
- **ProcessOpenStatementsService** can finalize “current window” statements **before the window ends**, and later orders in the same window create **new Open statements** for the same user+currency+periodStart. This can repeat, producing 3rd/4th/5th statements for the same window.
- **Window boundary math is off by one day** and can overlap windows (start=end day). This breaks the “26th → 25th” requirement and increases mis-bucketing risk.
- **Timezone handling in BaseOrderService** can assign orders to the wrong window around midnight if server timezone ≠ `SYSTEM_TIME_ZONE`.

## 2) Evidence (files/methods/queries)

**PaymentStatement uniqueness**
- Unique index exists **only for Open statements** and keyed by `userId, currency, interval, periodStart`:
  - `src/payment/entity/payment-statement.entity.ts` (`@Index('UX_ps_open_user_currency_interval_period', { synchronize: false })`)
  - `src/common/migrations/1769000000001-migration.ts` (`CREATE UNIQUE INDEX ... WHERE status = 'open'`)
- This **does not prevent duplicates across other statuses** (Finalized, Paid, etc.).

**Statement creation**
- Only creation path found in `BaseOrderService.getOrCreatePaymentStatement`:
  - `src/order/service/base-order.service.ts` (`getOrCreatePaymentStatement`, `execute`)
  - Uses `status = Open` and `periodStart` filter only.

**Monthly billing + retry pipeline**
- Monthly billing: `MonthlyBillingService.processBillingDay`
  - `src/billing/services/monthly-billing.service.ts`
  - Claims `Open` statements where `periodEnd = bizDate`.
- Retry: `PaymentRetryService.processRetryDay`
  - `src/billing/services/payment-retry.service.ts`
  - Calls `ProcessOpenStatementsService.execute` every retry day (26–28).
- `ProcessOpenStatementsService.execute`
  - `src/billing/services/process-open-statements.service.ts`
  - Claims `Open` statements where `periodEnd <= currentEnd`
  - `currentEnd` is **next month’s 25th** when run on the 26th–28th.

**Window computation**
- `BaseOrderService.getBillingWindow`:
  - `src/order/service/base-order.service.ts`
  - Uses `BILLING_PERIOD_END_DAY` as *start* day; does not use start day.
  - Uses `DateTime.fromJSDate(now).toISODate()` then parses in business TZ → can shift day.

**Invoice period interpretation**
- Stripe invoice line period uses `startOf('day')` / `endOf('day')`
  - `src/payment/services/stripe/create-invoice.ts` (`statementPeriodToUtcSeconds`)
  - So periodStart/periodEnd are inclusive boundaries.

## 3) Execution paths where duplicates can occur

### A) Retry path → Early finalization → new Open statement (repeatable)
1. **Feb 26 (retry day)**: `PaymentRetryService.processRetryDay` calls `ProcessOpenStatementsService.execute`.  
   - `currentEnd = Mar 25` (because day ≥ 26).  
   - It claims **all Open statements with `periodEnd <= Mar 25`**, which includes **current window** statements (ending Mar 25) that are **not yet due**.
2. Those statements are finalized (invoice created) by `BaseBillingService.processClaimedStatements`.
3. **Later on Feb 27**, user places another MonthlyBilling order.
   - `getOrCreatePaymentStatement` only looks for `status = Open`.
   - Existing statement is **Finalized**, so it **creates a new Open statement** with the **same periodStart**.
4. Retry job runs again (or next day), repeats the cycle.  
   ⇒ **Multiple statements for same user+currency+window**.

### B) Date boundary logic → overlapping windows
- With current logic, windows are **25th → 25th inclusive**.
- Orders on **Feb 25** go to a window starting **Feb 25**, ending **Mar 25**.
- Orders on **Feb 24** go to a window starting **Jan 25**, ending **Feb 25**.
- **Feb 25 is included in both windows** (inclusive end), causing **overlap** and mis-bucketing.  
  This can result in separate statements for adjacent windows that **both cover the 25th**.

### C) Concurrency (conditional on migration)
- If `UX_ps_open_user_currency_interval_period` was **not applied** in production, two concurrent order creates can insert two Open statements for same user+currency+periodStart because the pessimistic lock only applies if a row exists:
  - `src/order/service/base-order.service.ts` (`getOrCreatePaymentStatement`)
- The code handles duplicate-key errors **only if the unique index exists**.

## 4) Window math correctness (26th → 25th requirement)

Current code in `BaseOrderService.getBillingWindow` is inconsistent with the requirement:
- It uses **end day (25)** as the **start** day.
- It **does not use** `BILLING_PERIOD_START_DAY`.

Concrete examples in `SYSTEM_TIME_ZONE`:
- **Jan 24 12:00**  
  - Code window: **Dec 25 → Jan 25**  
  - Required window: **Dec 26 → Jan 25**  
  - ❌ Off by 1 day (Dec 25 wrongly included).
- **Jan 25 00:00**  
  - Code window: **Jan 25 → Feb 25**  
  - Required window: **Jan 26 → Feb 25**  
  - ❌ Off by 1 day (Jan 25 wrongly included).
- **Jan 26 00:00**  
  - Code window: **Jan 25 → Feb 25**  
  - Required window: **Jan 26 → Feb 25**  
  - ❌ Off by 1 day.

Timezone drift:
- `DateTime.fromJSDate(now).toISODate()` uses **server zone**, not `SYSTEM_TIME_ZONE`.  
  If server runs in UTC and business zone is New York, orders placed in late evening NY can be treated as next day, shifting them into the wrong window.

## 5) Root causes
- **Filtered unique index only for Open statements** (`status='open'`), allowing multiple statements per window once status changes.
- **ProcessOpenStatementsService** bills **current** (future-due) window during retry days because it uses `periodEnd <= currentEnd` with `currentEnd` set to **next month’s 25th**.
- **Billing window calculation** in `BaseOrderService` uses end day as start day and ignores start day.  
  Also uses server timezone, not business timezone, for date boundaries.

## 6) Fix recommendations (ranked)

### 1) Hard guarantee (DB-level)
- Add **unique constraint** on `(userId, currency, interval, periodStart)` **without status filter**, or introduce a `windowKey` (e.g., `YYYY-MM-26`) and make it unique.  
  This enforces **one statement per window per currency** across all statuses.
  - Migration example: `UNIQUE(userId, currency, interval, periodStart)`
- If you must allow “archived” duplicates, add a **soft-delete** column and unique constraint on `deletedAt IS NULL`.

### 2) Correct window computation
- Update `BaseOrderService.getBillingWindow` to use **start day = 26** and **end day = 25**:
  - Use `DateTime.fromJSDate(now, { zone: SYSTEM_TIME_ZONE })` to avoid timezone drift.
  - Compute:
    - `startCandidate = today.set({ day: startDay })`
    - `start = today >= startCandidate ? startCandidate : startCandidate.minus({ months: 1 })`
    - `end = start.plus({ months: 1 }).minus({ days: 1 })` (so 26→25 inclusive)
- Ensure periodStart/periodEnd are **non-overlapping**.

### 3) Stop early billing in retry flow
- In `ProcessOpenStatementsService.execute`, do **not** bill windows whose `periodEnd` is in the future.
  - Use `periodEnd <= bizDay` (or `= bizDay`) instead of `<= currentEnd`.
  - Alternatively, pass **previous window end** explicitly from `PaymentRetryService` using its `getBillingWindows()` helper.

### 4) Idempotent upsert for statement creation
- Replace the “read + insert” pattern with a **single upsert** on the unique key (now enforced).
- Or `SELECT ... WITH (UPDLOCK, HOLDLOCK)` on the key range before insert (SQL Server) to ensure no duplicates even without relying on exception handling.

### 5) Add a dedicated `windowKey`
- Store `windowKey` (e.g., `2026-01-26`) in `payment_statements` and use it for:
  - Unique constraint
  - Consistent selection across services
  - Clear reporting / debugging
