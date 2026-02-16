# Auto-Registered Kit Manual Registration Handling

# 1. Executive Summary

When kits are shipped, they are automatically registered to the correct account through the auto-registration system. However, some users attempt to manually register kits that have already been auto-registered to them.

Currently, this results in a blocking error (409 Conflict). This creates confusion and unnecessary friction.

This feature introduces a **graceful exemption flow**:

If a user attempts to manually register a kit that:

* Was auto-registered, and
* Was auto-registered to the same user

Then instead of throwing an error, the system will:

* Return a successful response
* Trigger a frontend modal
* Guide the user to their report history

This improves UX, reduces support friction, and aligns behavior with system automation.

---

# 2. Problem Statement

### Current Behavior

* User attempts to manually register a kit.
* If the kit is already registered → 409 Conflict.
* Even if it was auto-registered to the same user.

### Issues

* Creates confusion.
* Users believe something is broken.
* Increases support requests.
* Poor UX for a valid scenario.

---

# 3. Goals

### Primary Goal

Provide a user-friendly experience when a kit was already auto-registered to the same user.

### Secondary Goals

* Preserve strict protection when kit belongs to another user.
* Avoid leaking sensitive ownership information.
* Keep backend logic secure and deterministic.
* Keep frontend behavior simple and predictable.

---

# 4. Non-Goals

* We are not allowing re-registration of already registered kits.
* We are not changing auto-registration logic.
* We are not changing report history architecture.
* We are not enabling ownership transfer.

---

# 5. Functional Requirements

## 5.1 Backend Logic

When a manual registration request is made:

### Case A — Kit Not Registered

* Proceed with normal registration.
* Return success.

### Case B — Kit Registered

Evaluate:

```
IF registeredByAuto === true
AND registeredToUserId === requestingUserId
→ Exemption case
```

#### Exemption Case Behavior

* Return HTTP 200
* Include structured response flag:

  * `result: "ALREADY_REGISTERED_AUTO"`

#### Non-Exemption Case

* Return HTTP 409
* Error code: `KIT_ALREADY_REGISTERED`

---

## 5.2 API Contract

### Endpoint

`POST /kits/{kitId}/register`

---

### Exemption Response (HTTP 200)

```json
{
  "ok": true,
  "result": "ALREADY_REGISTERED_AUTO",
  "data": {
    "kitId": "DP000123",
    "registeredAt": "2026-02-16T14:10:00.000Z"
  }
}
```

---

### Already Registered (Non-Exemption) Response (HTTP 409)

```json
{
  "ok": false,
  "error": {
    "code": "KIT_ALREADY_REGISTERED",
    "message": "This kit has already been registered."
  }
}
```

---

# 6. Frontend Requirements

## 6.1 Modal Trigger

When response includes:

```
result === "ALREADY_REGISTERED_AUTO"
```

Frontend must display modal.

---

## 6.2 Modal Design Requirements

### Title

Kit already registered

### Body

This kit was automatically registered to your account after shipment. You don’t need to register it again.

### Primary CTA

View Report History

→ Routes user to Report History page

### Secondary CTA

Close

---

## 6.3 Modal Behavior

* Modal blocks page interaction until dismissed.
* Primary button navigates.
* Secondary button closes modal.
* No error toast should appear in this case.

---

# 7. User Experience Flow

### Flow 1 — Normal Registration

1. User enters kit ID.
2. Backend registers kit.
3. Success confirmation shown.

---

### Flow 2 — Auto-Registered (Same User)

1. User enters kit ID.
2. Backend detects:

   * Registered
   * Registered by auto
   * Same user
3. Backend returns 200 with result flag.
4. Frontend shows modal.
5. User clicks “View Report History”.
6. User navigates to reports.

---

### Flow 3 — Registered to Another User

1. User enters kit ID.
2. Backend detects registered to different user.
3. Returns 409 error.
4. Frontend shows standard error message.

---

# 8. Security Requirements

* Do not expose which user owns the kit.
* Do not expose internal registration metadata beyond timestamp.
* Same error message must be used whether:

  * Registered by manual flow
  * Registered to another user
* Prevent enumeration attacks via consistent response timing.

---

# 9. Acceptance Criteria

## Backend

* [ ] Exemption logic correctly checks both:

  * `registeredByAuto === true`
  * `registeredToUserId === requestingUserId`
* [ ] Exemption returns HTTP 200
* [ ] Non-exemption returns HTTP 409
* [ ] No ownership information leaked

## Frontend

* [ ] Modal appears only when `result === ALREADY_REGISTERED_AUTO`
* [ ] No error toast in exemption case
* [ ] Primary button routes to report history
* [ ] Modal works on mobile and desktop
* [ ] UI matches design system

---

# 10. Edge Cases

* Kit auto-registered but user account later changed → treat as non-exemption.
* Kit registered manually originally → 409.
* Kit ID invalid → 404.
* Race condition during registration → maintain transaction safety.

---

# 11. Risks

| Risk                                              | Mitigation                               |
| ------------------------------------------------- | ---------------------------------------- |
| Frontend misinterprets flag                       | Strict enum contract                     |
| Developer accidentally returns 200 for wrong case | Add unit + integration tests             |
| Future schema change breaks logic                 | Add test coverage around exemption logic |

---

# 12. Metrics for Success

* Reduction in “Kit already registered” support tickets.
* Reduction in 409 registration attempts.
* Increase in successful navigation to Report History from modal.

---

# 13. Technical Notes (Engineering)

* Implement logic in registration service layer.
* Add enum:

  ```
  enum KitRegistrationResult {
    SUCCESS,
    ALREADY_REGISTERED_AUTO
  }
  ```
* Ensure transactional consistency.
* Add unit tests for all 3 paths.
* Add integration test for API contract.

---
