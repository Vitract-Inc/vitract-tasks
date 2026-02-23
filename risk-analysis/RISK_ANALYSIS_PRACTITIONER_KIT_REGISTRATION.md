# Risk Analysis: Practitioner Client Kit Registration

**Component:** `src/components/practitioner/content/RegisterClientKit/index.tsx`

**Last Reviewed:** 2026-02-23

---

## Executive Summary

This document identifies potential risks and failure points in the practitioner client kit registration flow, including kit registration (Step 1) and health information completion (Step 2).

---

## Component Architecture

### Main Flow Components
1. **RegisterClientKit/index.tsx** - Main orchestrator with 3-step wizard
2. **RegisterKit.tsx** - Step 1: Kit registration form
3. **HealthInfo.tsx** - Step 2: Health questionnaire
4. **Submitted.tsx** - Step 3: Confirmation screen

### Key Dependencies
- `useUser` hook - Fetches practitioner user profile
- `useUserKit` hook - Fetches kit list (pagination)
- `useCategory` hook - Fetches health questionnaire categories
- `useResponseSseStore` - Real-time SSE state management for health info
- `KitRequest` - API service for kit operations
- `QuestionRequest` - API service for questionnaire operations

---

## CRITICAL RISKS - Kit Registration (Step 1)

### 1. **Kit ID Validation Failures**

**Risk Level:** 🔴 HIGH

**Location:** `RegisterKit.tsx` lines 8-17, schema validation

**Issue:**
- Complex regex validation for multiple kit formats (VT, BS, DP, 00, 11-99)
- Practitioner schema allows more prefixes than customer schema
- Length validation differs by prefix (8 chars for 00/DP, 9 for VT/BS)

**Failure Scenarios:**
```md
// Valid formats that might fail:
- "VT1234567" (9 chars, starts with VT)
- "00123456" (8 chars, starts with 00)
- "DP123456" (8 chars, starts with DP)
- "11234567" (8 chars, practitioner-only prefix)

// Edge cases:
- Mixed case input (handled by .toUpperCase())
- Pasted values with hyphens (handled by .replace("-", ""))
- Special characters in kit ID
- Whitespace in pasted values
```

**Root Causes:**
1. Schema regex: `/^(VT|BS|DP|00|11|22|33|44|55|66|77|88|99)\d[a-zA-Z0-9-]+$/`
2. OTP input only accepts 8 characters but some kits need 9
3. No server-side validation feedback before submission

**Impact:**
- Users cannot register valid kits
- Confusing error messages
- Support ticket escalation

**Mitigation:**
- Add real-time kit format detection
- Show format-specific hints based on first 2 characters
- Server-side validation should return specific error codes

---

### 2. **Date Validation Edge Cases**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 168-169, schema lines 39-44

**Issue:**
```javascript
const maxDate = new Date(); // Today
const minDate = new Date(new Date().getFullYear() - 2, 0, 1); // 2 years ago
```

**Failure Scenarios:**
- Timezone mismatches (client vs server)
- Date picker allows future dates if maxDate calculation is off
- Samples collected exactly 2 years ago might fail boundary check
- Date serialization issues when sending to API (ISO string conversion)

**Root Cause:**
- Client-side date validation only
- No timezone normalization
- Boundary conditions not clearly defined

**Impact:**
- Valid samples rejected
- Data inconsistency between client and server

---

### 3. **Kit Already Registered Conflict**

**Risk Level:** 🔴 HIGH

**Location:** `RegisterKit.tsx` lines 142-149

**Issue:**
```javascript
if (isAlreadyRegisteredConflict(error)) {
  // Shows generic error, doesn't explain WHO registered it
  toastError(KIT_REGISTRATION_CONFLICT_MESSAGE);
  return;
}
```

**Failure Scenarios:**
- Kit registered by another practitioner
- Kit registered by customer directly
- Kit registered in different environment (staging vs production)
- No way to request transfer or override

**Root Cause:**
- HTTP 409 conflict with error code "KIT_ALREADY_REGISTERED"
- No additional context in error response
- No resolution workflow

**Impact:**
- Practitioners blocked from registering legitimate kits
- Customer confusion if kit was pre-registered
- Manual support intervention required

---

### 4. **Auto-Registration Exemption Modal**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 128-138, 246-250

**Issue:**
```javascript
if (outcome.type === "already_registered_auto") {
  setIsExemptionModalOpen(true);
  return; // Doesn't proceed to step 2
}
```

**Failure Scenarios:**
- Modal opens but user doesn't understand what "already registered auto" means
- User closes modal and loses context
- No clear path forward after modal dismissal
- Form state not preserved if user navigates away

**Root Cause:**
- Unclear UX for auto-registration scenario
- Backend returns HTTP 200 with special result code
- No explanation of what auto-registration means

**Impact:**
- User confusion
- Abandoned registration flows
- Duplicate support tickets

---

### 5. **Lock Status Logic Complexity**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 99-105

**Issue:**
```javascript
lockStatus:
  userHook.data &&
  userHook.data.clientPractitioners.some(
    (eq) => eq.reportAccess === PractitionerAccessStatus.GRANTED
  )
    ? LockStatus.LOCKED
    : LockStatus.UNLOCKED,
```

**Failure Scenarios:**
- `userHook.data` is undefined/null
- `clientPractitioners` array is empty or undefined
- Race condition: user data not loaded when form submits
- Logic doesn't account for multiple practitioners with different access levels

**Root Cause:**
- Complex conditional logic without null safety
- Depends on user data being fully loaded
- No fallback for edge cases

**Impact:**
- Incorrect lock status set
- Report access issues downstream
- Data integrity problems

---

### 6. **Update vs Create Mode Confusion**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 54-59, 108-110

**Issue:**
```javascript
const isUpdatingRegisteredKit =
  userHook.data &&
  userHook.data.practitionerKit &&
  userHook.data.practitionerKit.status === KitStatus.REGISTERED &&
  update &&
  step === 1;

if (isUpdatingRegisteredKit) {
  data.kitId = userHook.data.practitionerKit.id; // Adds kit ID for update
}
```

**Failure Scenarios:**
- `update` prop is true but `practitionerKit` is null
- Kit status changed between page load and submission
- Initial values populated from old kit data
- User edits kit ID but backend uses `kitId` field for update

**Root Cause:**
- Complex boolean logic for mode detection
- State synchronization between parent and child components
- No explicit update confirmation

**Impact:**
- Wrong kit updated
- Data overwritten unintentionally
- User confusion about what's being edited

---

## CRITICAL RISKS - Health Information (Step 2)

### 7. **SSE Connection Failures**

**Risk Level:** 🔴 CRITICAL

**Location:** `HealthInfo.tsx` lines 97-131

**Issue:**
```javascript
useEffect(() => {
  const initializeSSE = async () => {
    // Complex initialization with multiple async steps
    joinKit(kitNumber);
    await new Promise((r) => setTimeout(r, 150));
    await refreshData(kitNumber);
    await new Promise((r) => setTimeout(r, 250));
    if (!initializationRef.current.hasFetchedAnswers && isCacheEmpty(kitNumber)) {
      await fetchAnswer(kitNumber);
    }
  };
  const t = setTimeout(initializeSSE, 200);
}, [kitNumber, currentKitId, responses, joinKit, refreshData]);
```

**Failure Scenarios:**
- SSE connection fails (network issues, CORS, auth)
- Server doesn't support SSE or endpoint is down
- Multiple rapid re-renders cause connection spam
- Race conditions between joinKit, refreshData, fetchAnswer
- Timeout values (150ms, 250ms, 200ms) too aggressive for slow networks
- `initializationRef` prevents re-initialization even when needed

**Root Causes:**
1. No error handling for SSE connection failures
2. Hard-coded timeouts without retry logic
3. Complex initialization sequence with hidden dependencies
4. useEffect dependency array includes functions that change on every render
5. No connection status indicator for users

**Impact:**
- Health info not loaded from backend
- Users see empty questionnaire even if they filled it before
- Data loss if users re-answer questions
- Silent failures - no error shown to user
- Multiple concurrent SSE connections drain resources

**Mitigation Required:**
- Add connection status UI
- Implement exponential backoff retry
- Add error boundaries
- Fallback to REST API if SSE fails

---

### 8. **Answer Fetching Race Conditions**

**Risk Level:** 🔴 HIGH

**Location:** `HealthInfo.tsx` lines 79-92, 118-120

**Issue:**
```javascript
const fetchAnswer = async (kitId: string) => {
  try {
    setIsAnswerLoading(true);
    setAnswerError(null);
    const response = await questionRequest.fetchAnswers(kitId, 0); // categoryId = 0
    if (response.data.status === false) return; // Silent failure
    const healthAnswer = transformBackendResponseToCache(response.data.questionnaireResponses, kitId);
    await addResponsesFromBackend(kitId, healthAnswer);
  } catch (error) {
    setAnswerError(error as any);
  } finally {
    setIsAnswerLoading(false);
  }
};
```

**Failure Scenarios:**
- `categoryId: 0` might not return all categories
- `response.data.status === false` returns silently - user sees empty form
- `transformBackendResponseToCache` throws error - not caught
- `addResponsesFromBackend` is async but error not handled
- Multiple calls to fetchAnswer if kitNumber changes rapidly
- `hasFetchedAnswers` flag never resets if fetch fails

**Root Causes:**
1. Fetches only categoryId 0 instead of all categories
2. Silent failure on `status === false`
3. No validation of response data structure
4. Async operations without proper error handling
5. State management race conditions

**Impact:**
- Previously entered answers not loaded
- Users re-enter data (poor UX)
- Data inconsistency between cache and backend
- Duplicate submissions

---

### 9. **Response Store Synchronization Issues**

**Risk Level:** 🔴 HIGH

**Location:** `HealthInfo.tsx` lines 66-67, store lines 264-266

**Issue:**
```javascript
const kitNumber = userHook?.data?.practitionerKit?.kitNumber;
const disabled = setDisabledStatus(responses, kitNumber);

// setDisabledStatus function:
export function setDisabledStatus(responses: ResponseInterface[], kitId: string) {
  if (kitId && responses.length) {
    const disabled = responses.some(
      (rs) => rs.kitId === kitId && rs.completed !== true
    );
    return disabled;
  }
  return true; // Disabled if no kitId or no responses
}
```

**Failure Scenarios:**
- `kitNumber` is undefined → button always disabled
- `responses` array contains data from different kits
- SSE updates responses but component doesn't re-render
- `completed` flag not set correctly by SSE events
- User fills all categories but button stays disabled
- Race condition: responses updated after disabled calculation

**Root Causes:**
1. No null safety for kitNumber
2. Responses array not filtered by kitId before checking
3. SSE store updates don't trigger re-render reliably
4. `completed` flag logic unclear
5. No loading state differentiation

**Impact:**
- Submit button permanently disabled
- Users cannot proceed even after completing all questions
- Support escalation required
- Abandoned workflows

---

### 10. **Question Submission Payload Transformation**

**Risk Level:** 🟡 MEDIUM

**Location:** `HealthInfo.tsx` lines 145-178

**Issue:**
```javascript
async function handleSubmit() {
  const payload: any[] = []; // Type safety lost
  for (const rs of responses.filter((rs) => rs.kitId === kitNumber)) {
    for (const qs of rs.questionsResponse) {
      const answerPayload = qs.answers.map((ans) => ({
        ...ans,
        selectedOptions: ans.selectedOptions?.toString(), // Array to string
      }));
      payload.push({ ...qs, answers: answerPayload });
    }
  }

  await questionRequest.submitQuestions({ responses: payload });
  await kitRequest.updatePractitionerKitStatus(kitNumber, {
    status: KitStatus.AWAITNG_SAMPLE,
    date: new Date(),
    submitted: true,
    healthInfoCompleted: HealthInfoStatus.YES,
  });
}
```

**Failure Scenarios:**
- `selectedOptions` is array but converted to string - data loss
- Empty responses array passes validation but submits nothing
- `kitNumber` is undefined - API call fails
- First API call succeeds but second fails - inconsistent state
- No validation of answer completeness
- `any[]` type allows invalid data structure

**Root Causes:**
1. Lossy data transformation (array → string)
2. No transaction handling for multiple API calls
3. No payload validation before submission
4. Type safety bypassed with `any`
5. No rollback on partial failure

**Impact:**
- Data corruption (multi-select answers lost)
- Inconsistent database state
- Failed submissions with no error message
- Users think they submitted but data not saved

---

### 11. **Category Loading and Initialization**

**Risk Level:** 🟡 MEDIUM

**Location:** `HealthInfo.tsx` lines 303-363

**Issue:**
```javascript
const Category = ({ userHook }: CategoryProps) => {
  const { data, isLoading, error } = useCategory();

  if (data && userData) {
    return (
      <>
        {data.map((category) => (
          <div onClick={() => handleQuestion(category)}>
            {/* Category UI */}
          </div>
        ))}
      </>
    );
  } else if (isLoading || userHook.isLoading) {
    return <ComponentLoader />;
  } else if (error || userHook.error) {
    return <ErrorComponent error={error} />;
  } else {
    return <p className="text-center">No category available</p>;
  }
};
```

**Failure Scenarios:**
- `useCategory` hook fails to fetch - shows "No category available"
- `data` is empty array - shows "No category available"
- Network timeout - stuck in loading state
- `userData` is undefined but `data` exists - shows nothing
- Error component doesn't provide retry option

**Root Causes:**
1. No retry mechanism for failed category fetch
2. Empty data treated same as error
3. No distinction between loading and error states
4. Depends on `useCategory` hook with `revalidateOnMount: false`
5. No cache invalidation strategy

**Impact:**
- Users cannot access questionnaire
- No way to recover without page refresh
- Poor error messaging
- Support tickets

---

### 12. **Kit Status Transition Logic**

**Risk Level:** 🟡 MEDIUM

**Location:** `index.tsx` lines 59-77

**Issue:**
```javascript
useEffect(() => {
  if (
    userHook.data &&
    userHook.data.practitionerKit &&
    !reset &&
    !update &&
    !userHook.data?.practitionerKit?.submitted
  ) {
    handleStep(2); // Auto-advance to health info
  } else if (
    userHook.data &&
    userHook.data.practitionerKit &&
    !reset &&
    !update &&
    userHook.data?.practitionerKit?.submitted
  ) {
    handleStep(3); // Auto-advance to submitted
  }
}, [userHook.data]);
```

**Failure Scenarios:**
- User manually navigates back - auto-advanced forward again
- `reset` or `update` flags not set correctly
- `practitionerKit.submitted` is undefined (not false) - wrong step shown
- Race condition: `userHook.data` updates multiple times
- No validation of kit status before auto-advancing
- Infinite loop if `handleStep` triggers data reload

**Root Causes:**
1. Boolean logic doesn't handle undefined/null
2. No check for kit status (REGISTERED vs AWAITING_SAMPLE)
3. useEffect runs on every data change
4. No dependency on `reset` and `update` in useEffect
5. Auto-navigation overrides user intent

**Impact:**
- Users stuck in wrong step
- Cannot edit previous steps
- Confusing navigation behavior
- Data entry errors

---

## ADDITIONAL RISKS

### 13. **Network and Infrastructure Failures**

**Risk Level:** 🔴 HIGH

**Common Failure Scenarios:**
- API endpoint timeouts (no timeout configuration visible)
- CORS issues in production vs development
- Authentication token expiration during multi-step flow
- Rate limiting on API endpoints
- Server 500 errors with no retry logic
- Network disconnection during SSE session

**Impact:**
- Complete workflow failure
- Data loss
- User frustration
- No offline capability

---

### 14. **Browser Compatibility Issues**

**Risk Level:** 🟡 MEDIUM

**Potential Issues:**
- SSE not supported in older browsers
- DatePicker component rendering issues
- OTP input focus behavior inconsistent
- SessionStorage/LocalStorage quota exceeded
- Browser back button breaks state

**Impact:**
- Feature unavailable for some users
- Inconsistent UX across browsers

---

## RECOMMENDATIONS

### Immediate Actions (High Priority)

1. **Add SSE Error Handling**
   - Implement connection status indicator
   - Add fallback to REST API
   - Implement retry logic with exponential backoff

2. **Fix Submit Button Logic**
   - Add null safety for kitNumber
   - Filter responses by kitId before checking completion
   - Add loading state differentiation

3. **Improve Kit ID Validation**
   - Make OTP input dynamic length (8 or 9 chars)
   - Add real-time format hints
   - Better error messages

4. **Add Transaction Handling**
   - Wrap multiple API calls in try-catch
   - Implement rollback on partial failure
   - Add optimistic UI updates

### Medium Priority

5. **Enhance Error Recovery**
   - Add retry buttons on error states
   - Preserve form state on navigation
   - Add error telemetry

6. **Improve Data Validation**
   - Validate payload before submission
   - Add TypeScript strict mode
   - Server-side validation feedback

7. **Fix Auto-Navigation Logic**
   - Add explicit user confirmation
   - Check kit status before advancing
   - Prevent infinite loops

### Long-term Improvements

8. **Add Monitoring**
   - Track SSE connection success rate
   - Monitor API response times
   - Alert on high error rates

9. **Improve UX**
   - Add progress indicators
   - Better loading states
   - Clearer error messages
   - Offline support

10. **Code Quality**
    - Reduce complexity in useEffect hooks
    - Extract business logic from components
    - Add comprehensive unit tests
    - Add integration tests for multi-step flow

---

## MONITORING SUGGESTIONS

### Key Metrics to Track

1. **Kit Registration Success Rate**
   - Track by kit prefix type
   - Monitor validation failures
   - Track 409 conflicts

2. **Health Info Completion Rate**
   - SSE connection success rate
   - Answer fetch success rate
   - Submission success rate
   - Time to complete questionnaire

3. **Error Rates**
   - API errors by endpoint
   - Client-side errors by component
   - Network failures
   - Timeout occurrences

4. **User Behavior**
   - Step abandonment rates
   - Back button usage
   - Form field errors
   - Support ticket correlation

---

**Document Version:** 1.0
**Last Updated:** 2026-02-23
**Next Review:** Quarterly or after major changes

