# **PRD — Unified Stripe Checkout Session Generation + Dynamic Price Configuration**
---

## **1. Objective**

Create a **unified, reusable Stripe Checkout session generator** that works for both:

1. **Authenticated platform users**, and
2. **Unauthenticated website visitors**

…using a **database-driven Stripe price configuration table** rather than environment variables or manually created Stripe Payment Links.

The goal is to **centralize price lookup**, remove `.env` price logic, eliminate marketing-created ad-hoc links, and provide a single entry point for creating all checkout sessions.

---

## **2. Problems This Solves**

* Price IDs are currently stored in `.env` and duplicated across flows.
* Marketing manually creates Stripe Payment Links, leading to inconsistent handling.
* Checkout logic differs between authenticated and guest flows.
* Hardcoded branch logic for `kitType` + `currency` combinations limits extensibility.

---

## **3. Deliverables**

### **D1. New Database Table: `stripe_prices`**

Stores Stripe price configurations for dynamic lookup.

#### **Columns (Engineering Specification)**

| Column            | Type                    | Description                                                     |
| ----------------- | ----------------------- | --------------------------------------------------------------- |
| `id`              | UUID                    | PK                                                              |
| `kitType`         | varchar(50)             | `gut-scan`, `deep-gut`, `deep-gut-plus` (enum stored as string) |
| `paymentType`     | varchar(50)             | `PLATFORM_ORDER`, `WEBSITE_ORDER`, `KIT_REPLACEMENT_ORDER`      |
| `currency`        | varchar(10)             | `USD` or `CAD`                                                  |
| `stripePriceId`   | varchar(255)            | Stripe Price ID                                                 |
| `stripeProductId` | varchar(255), nullable  | Optional reference                                              |
| `amountMinor`     | bigint                  | Original price in minor amount                                  |
| `mode`            | varchar(20)             | `payment` or `subscription`                                     |
| `interval`        | varchar(20), nullable   | For subscription support (future)                               |
| `isActive`        | boolean                 | Soft-enable new pricing                                         |
| `description`     | nvarchar(255), nullable | Admin-facing description                                        |

#### **Constraints**

* Unique index on: **(kitType, paymentType, currency)** where `isActive = TRUE`.

---

## **4. Unified Checkout Session Service**

Create a new backend service function:

```ts
createCheckoutSession(params: CreateCheckoutSessionParams)
```

### **4.1 Inputs**

| Parameter         | Required?                 | Description                                                    |
| ----------------- | ------------------------- | -------------------------------------------------------------- |
| `kitType`         | Yes                       | Identifies the type of kit                                     |
| `currency`        | Yes                       | USD/CAD                                                        |
| `paymentType`     | Yes                       | Where the checkout originates (platform, website, replacement) |
| `referenceId`     | Yes                       | Internally generated identifier tied to the order              |
| `quantity`        | No                        | Defaults to `1`                                                |
| `isAuthenticated` | Yes                       | Boolean switch for flow type                                   |
| `userId`          | Required if authenticated | Needed to retrieve/create Stripe Customer                      |
| `customerEmail`   | Only for guest            | Optional; can be omitted                                       |
| `customerName`    | Only for guest            | Optional                                                       |
| `successUrl`      | Yes                       | Redirect URL                                                   |
| `cancelUrl`       | Yes                       | Redirect URL                                                   |

### **4.2 Behavior**

#### **Authenticated Flow**

1. Fetch user by `userId`.
2. Ensure Stripe Customer exists:

   * If none exists, create one and store `stripeId`.
3. Use user's name/email for Checkout pre-fill.
4. Lookup the correct price from `stripe_prices`.
5. Create a Checkout session using:

   * `customer` (Stripe customer ID)
   * price from DB
   * standard metadata (kitType, referenceId, paymentType, currency)
6. Return session URL + session ID.

#### **Guest Flow**

1. No email required during session creation.
2. Lookup price from DB using the same `kitType + paymentType + currency` combination.
3. Create a Checkout session with:

   * **No `customer`**
   * **No `customer_email`**
   * Stripe will collect email/address at checkout.
4. Include `referenceId` + metadata for linking.

---

## **5. Country Handling Requirements**

* Allowed countries must be restricted to: **US** and **CA**.
* If Checkout requires address collection, use:

```ts
shipping_address_collection: {
  allowed_countries: ['US', 'CA']
}
```

* Users must be able to change the country during checkout.
* System must NOT enforce a default country.

---

## **6. Functional Requirements**

### **FR-1: Price Lookup**

* `createCheckoutSession()` must always fetch the price using the database table.
* If no matching price is found → throw a structured error.

### **FR-2: Unified Logic**

* All flows must call the same checkout generation function.
* No duplicated logic for authenticated vs guest.

### **FR-3: No Email Required for Guest**

* Email should not be mandatory for website-based checkouts.

### **FR-4: Metadata Inclusion**

The session must include standardized metadata:

```ts
{
  referenceId,
  kitType,
  paymentType,
  currency,
}
```

### **FR-5: Strong Input Validation**

* Validate enum strings for `kitType`, `paymentType`, `currency`.
* Validate required params based on `isAuthenticated`.

---

## **7. Non-Functional Requirements**

### **Performance**

* Price lookup must be O(1) via indexed query.
* Checkout session must be created < 300ms.

### **Reliability**

* Must not depend on `.env` price IDs anywhere after launch.

### **Scalability**

* Adding new kit types or currencies must not require code changes.

### **Maintainability**

* Engineers should be able to enable/disable prices by toggling `isActive`.

---

## **8. Acceptance Criteria**

### **AC-1**

Given a valid authenticated request,
When `createCheckoutSession` is called,
Then a Stripe session URL is returned using the price pulled from DB.

### **AC-2**

Given a valid guest checkout request,
When `createCheckoutSession` is called without an email,
Then Stripe Checkout collects email, and the session is successfully created.

### **AC-3**

Given an invalid price configuration (missing DB row),
The service must return a descriptive error.

### **AC-4**

No flow should reference `.env` price variables after implementation.

### **AC-5**

Marketing-created Stripe payment links are no longer required or used.

---

## **9. Deliverables Checklist (Engineering)**

* [ ] Create migration for `stripe_prices` table
* [ ] Add TypeORM entity
* [ ] Seed initial prices (from current env or Stripe)
* [ ] Implement `createCheckoutSession()` service
* [ ] Update internal platform order flow
* [ ] Add guest checkout integration endpoint
* [ ] Remove price lookup logic from `.env`
* [ ] Add internal documentation for engineers

---
