# **PRD — Step 2: Replacement Kit Request & Admin-Initiated Checkout**

### **Engineering Task Document**

**Version:** v1.0
**Depends on:** Step 1 — Unified Checkout Session Service
**Scope:** Backend + API contracts for Admin & Practitioner platforms
**Out of Scope:** Webhooks, order creation, fulfillment, refunds, email sending logic

---

## **1. Objective**

Implement a **Replacement Kit Request system** that allows an **admin** to initiate a paid replacement kit request for either:

* a **Practitioner** (authenticated user), or
* a **Client** (non-authenticated, email-based)

using the **Unified Checkout Session Service** introduced in Step 1.

A replacement kit request is **not an order**.
An order will be created **only after payment**, which is handled in a later step.

---

## **2. Key Principles**

1. A replacement kit request is a **payment request**, not an order.
2. Every request has a unique `referenceId`.
3. Payment is completed via a **Stripe Checkout link**, generated on demand.
4. **Client details are always provided**, regardless of target type.
5. If the target is a practitioner, the request is **linked to that practitioner**.
6. Address information is **mandatory** at request creation.
7. Replacement kit requests are **payable exactly once**.

---

## **3. Supported Target Types**

### **TargetType = PRACTITIONER**

* Admin creates a request on behalf of a practitioner.
* Admin provides:

  * Client details (name + email)
  * Shipping address
  * Quantity
  * Practitioner ID
* Checkout:

  * Authenticated flow
  * Uses practitioner’s Stripe customer
* Practitioner:

  * Sees the request in their platform
  * Can pay directly from the dashboard

---

### **TargetType = CLIENT**

* Admin creates a request directly for a client.
* Admin provides:

  * Client details (name + email)
  * Shipping address
  * Quantity
* Checkout:

  * Guest (non-authenticated) flow
* Client:

  * Receives checkout link via email only
  * No platform access required

---

## **4. Database Model**

### **Table: `kit_replacement_requests`**

This table stores **pending and completed replacement kit payment requests**.

### **Schema**

| Column             | Type                    | Description                                       |
| ------------------ | ----------------------- | ------------------------------------------------- |
| `id`               | UUID                    | Primary key                                       |
| `referenceId`      | varchar(100), unique    | Used to link checkout session                     |
| `targetType`       | varchar(30)             | `PRACTITIONER` or `CLIENT`                        |
| `practitionerId`   | UUID, nullable          | Required if target = PRACTITIONER                 |
| `firstName`        | varchar(100)            | Client first name                                 |
| `lastName`         | varchar(100)            | Client last name                                  |
| `email`            | varchar(255)            | Client email                                      |
| `kitType`          | varchar(50)             | Kit type                                          |
| `currency`         | varchar(10)             | USD / CAD                                         |
| `quantity`         | int                     | Number of kits                                    |
| `country`          | varchar(50)             | US / CA                                           |
| `addressLineOne`   | varchar(255)            | Required                                          |
| `addressLineTwo`   | varchar(255), nullable  | Optional                                          |
| `city`             | varchar(100)            | Required                                          |
| `state`            | varchar(100)            | Required                                          |
| `postalCode`       | varchar(20)             | Required                                          |
| `status`           | varchar(30)             | `PENDING_PAYMENT`, `PAID`, `CANCELLED`, `EXPIRED` |
| `paymentUrl`       | nvarchar(max), nullable | Cached checkout link                              |
| `createdByAdminId` | UUID                    | Admin who created request                         |
| `createdAt`        | datetime                | Timestamp                                         |
| `updatedAt`        | datetime                | Timestamp                                         |

---

## **5. Replacement Kit Request Creation**

### **Service**

`ReplacementKitRequestService.create()`

### **Input DTO**

```ts
CreateReplacementKitRequestDto {
  targetType: 'PRACTITIONER' | 'CLIENT';

  practitionerId?: string; // required if PRACTITIONER

  firstName: string;
  lastName: string;
  email: string;

  kitType: string;
  currency: 'USD' | 'CAD';
  quantity: number;

  address: {
    country: 'US' | 'CA';
    addressLineOne: string;
    addressLineTwo?: string;
    city: string;
    state: string;
    postalCode: string;
  };
}
```

### **Validation Rules**

* `firstName`, `lastName`, `email` → always required
* `address` → always required
* `quantity >= 1`
* If `targetType = PRACTITIONER` → `practitionerId` is required
* If `targetType = CLIENT` → `practitionerId` must be null

---

## **6. Checkout Link Generation**

### **Service**

`ReplacementKitCheckoutService.createCheckoutLink(requestId)`

### **Behavior**

1. Fetch replacement kit request.
2. Validate:

   * Status = `PENDING_PAYMENT`
3. Call **Unified Checkout Session Service** with:

#### **Common Params**

* `referenceId`
* `kitType`
* `currency`
* `quantity`
* `paymentType = KIT_REPLACEMENT_ORDER`

#### **Practitioner Target**

* `isAuthenticated = true`
* `userId = practitionerId`

#### **Client Target**

* `isAuthenticated = false`
* `customerEmail = email`

4. Save returned checkout URL to `paymentUrl`.
5. Return checkout URL.

---

## **7. API Endpoints**

### **Admin APIs**

#### **POST /admin/replacement-kit-requests**

Creates a replacement kit request.

#### **POST /admin/replacement-kit-requests/:id/checkout**

Generates checkout URL.

---

### **Practitioner APIs**

#### **GET /practitioner/replacement-kit-requests**

Returns requests linked to authenticated practitioner.

#### **POST /practitioner/replacement-kit-requests/:id/pay**

Generates checkout URL for practitioner payment.

---

## **8. Frontend Contracts**

### **Admin Platform**

* Admin form to create replacement kit request
* Fields:

  * Target type
  * Practitioner (if applicable)
  * Client name & email
  * Full shipping address
  * Quantity
* Admin does **not** pay

---

### **Practitioner Platform**

* Table listing replacement kit requests
* Each row shows:

  * Client name
  * Quantity
  * Status
  * Pay button
* Clicking Pay redirects to Stripe Checkout

---

### **Client Flow**

* Client receives payment link via email
* No dashboard access
* Payment only via Stripe Checkout

---

## **9. Functional Requirements**

### **FR-1**

Replacement kit requests must exist independently of orders.

### **FR-2**

Checkout sessions must be created via the Unified Checkout Session Service.

### **FR-3**

Replacement kit requests must be payable exactly once.

### **FR-4**

Client details must always be present, regardless of target type.

### **FR-5**

Practitioners must only see requests linked to them.

---

## **10. Non-Functional Requirements**

* Requests must be immutable once paid.
* Checkout generation must be idempotent.
* No `.env` price logic allowed.
* All pricing must come from the Stripe price table.

---

## **11. Acceptance Criteria**

* Admin can create replacement kit requests for both target types.
* Address is always required and stored.
* Practitioner can see and pay for their requests.
* Client-only requests require no authentication.
* Checkout session generation uses correct authenticated/guest flow.
* No order is created during this step.
