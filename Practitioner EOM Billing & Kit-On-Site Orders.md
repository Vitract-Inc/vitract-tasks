# Task Specification: Practitioner EOM Billing & Kit-On-Site Orders

## Overview

We are extending our ordering and payment system to support **End-of-Month (EOM) billing for practitioners** and a **Kit-on-Site order type** with quantity limits. We also need to track **amounts paid and promo codes** across all flows.

---

## 1. Data Model Changes

### Orders Table

Add the following fields:

* `currency` — string (e.g., "usd", "cad").
* `amountSubtotal` — bigint (pre-discount).
* `amountDiscount` — bigint.
* `amountTax` — bigint.
* `amountTotal` — bigint (final paid amount).
* `promotionCode` — string, nullable (human code like `SAVE20`).
* `promotionCodeId` — string, nullable (Stripe promotion\_code id).
* `invoiceId` — string, nullable (Stripe invoice id).
* `invoicingMode` — enum: `EOM`, `Immediate` (default to `EOM` for practitioners).
* Keep `quantity`, but enforce Kit-on-Site limit (≤20).

### Transactions Table

Add/update:

* `amount` — bigint.
* `currency` — string.
* `stripeInvoiceId` — string, nullable.
* `stripePaymentIntentId` — string, nullable.
* `promotionCode` / `promotionCodeId` — nullable.
* `paidAt` — datetime, nullable.

### PaymentStatement Table (new)

Fields:

* `id` — uuid.
* `userId` — practitioner’s user id.
* `interval` — enum: `monthly`, `yearly` (default monthly).
* `currency` — string.
* `periodStart` — datetime.
* `periodEnd` — datetime.
* `stripeInvoiceId` — string, nullable.
* `status` — enum: `open`, `finalized`, `paid`, `payment_failed`.
* Totals: `amountSubtotal`, `amountDiscount`, `amountTax`, `amountTotal`.
* `paidAt` — datetime, nullable.
* `createdAt`, `updatedAt`.

---

## 2. Order Creation Flow

* When a practitioner places an order:

  1. Determine **currency** from selected kit price.
  2. Compute **statement period**:

     * Monthly: `periodStart = 26th 00:00:00 prev month`, `periodEnd = 25th 23:59:59 current month`.
  3. **Upsert** an open `PaymentStatement` for `(userId, currency, interval, period window)`.
  4. Attach the order to that statement.
  5. Create a **Stripe Invoice Item** for the practitioner’s Customer:

     * Amount = unit price × quantity.
     * Metadata = `{orderReferenceId, kitType, quantity, country}`.
     * Apply promotion code if provided.
  6. Mark order `status = PendingInvoice`.
  7. Create a `Transaction` with `status = Pending`.

* For Kit-on-Site orders: **reject** if `quantity > 20`.

---

## 3. Billing & Retry Logic

### On the 25th (billing day)

* Job runs for all open statements whose `periodEnd = today`.
* For each statement:

  * Create a **Stripe Invoice** from attached invoice items:

    * `collection_method: charge_automatically`.
    * Default payment method from `PaymentMethod.isDefault`.
  * Finalize the invoice.
  * Update statement: `stripeInvoiceId`, `status = finalized`.
  * Attach `invoiceId` to linked orders.

### On the 26th & 27th (retries)

* For failed invoices:

  * Retry payment by calling `pay` again.
* If still failing after 27th:

  * Set statement `status = payment_failed`.
  * Set orders `status = PaymentFailed` (or `Dunning`).
  * Transactions → `Failed`.

---

## 4. Webhook Handling (Source of Truth)

* `invoice.payment_succeeded`:

  * Update statement: `status = paid`, `paidAt`, totals.
  * Update all orders in statement: set `Paid`, fill amounts, currency, promo code, `invoiceId`, `paidAt`.
  * Update transactions: `Successful`, fill amounts, invoice id, `paidAt`.

* `invoice.payment_failed`:

  * Update statement: `payment_failed`.
  * Transactions → `Failed`.

---

## 5. Reporting & Queries

* **Per Order**: exact paid amounts + applied promo.
* **Per Transaction**: reconciliation with Stripe invoice/payment intent ids.
* **Per Statement**: roll-up totals per practitioner, per month, per currency.

---

## 6. Acceptance Criteria

* Orders placed by practitioners **accumulate** into their open monthly statement until the 25th.
* On the 25th, all statements are finalized into invoices and charged.
* Orders created after the 25th run are attached to the **next month’s statement**.
* Kit-on-Site orders **cannot exceed 20 quantity**.
* Webhooks correctly update orders, transactions, and statements.
* Retry logic executes on the 26th and 27th; unpaid statements remain `payment_failed`.
