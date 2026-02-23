# Risk Analysis: Customer Kit Registration

**Component:** `src/components/customer/content/Overview/index.tsx`

**Last Reviewed:** 2026-02-23

---

## Executive Summary

This document identifies potential risks and failure points in the customer kit registration flow, including kit registration (Step 1) and health information completion (Step 2). The customer flow differs from the practitioner flow in several key ways: no SSE (uses sessionStorage), no client name field, different schema validation, and skip functionality.

---

## Component Architecture

### Main Flow Components
1. **Overview/index.tsx** - Main orchestrator with 3-step wizard
2. **RegisterKit.tsx** - Step 1: Kit registration form
3. **HealthInfo.tsx** - Step 2: Health questionnaire
4. **Submitted.tsx** - Step 3: Confirmation screen

### Key Dependencies
- `useUser` hook - Fetches customer user profile
- `useUserKit` hook - Fetches kit information
- `useCategory` hook - Fetches health questionnaire categories
- `useResponseStore` - SessionStorage-persisted state management
- `KitRequest` - API service for kit operations
- `QuestionRequest` - API service for questionnaire operations

### Key Differences from Practitioner Flow
- **No SSE**: Uses `useResponseStore` (sessionStorage) instead of `useResponseSseStore`
- **No Client Name**: Only kit ID and date required
- **Stricter Schema**: Only allows VT, BS, DP, 00 prefixes (no 11-99)
- **Skip Functionality**: "Skip all for now" button available
- **Different Auto-Navigation**: Uses `KitStatus` enum checks

---

## CRITICAL RISKS - Kit Registration (Step 1)

### 1. **Kit ID Validation Failures**

**Risk Level:** 🔴 HIGH

**Location:** `RegisterKit.tsx` lines 8-17, `create-kit.schema.ts`

**Issue:**
- Customer schema more restrictive than practitioner schema
- Regex: `/^(VT|BS|DP|00)\d[a-zA-Z0-9-]+$/`
- Length validation: 8 chars for 00/DP, 9 chars for VT/BS
- OTP input limited to 8 characters

**Failure Scenarios:**
```javascript
// Valid formats that might fail:
- "VT1234567" (9 chars) - OTP input only accepts 8
- "BS1234567" (9 chars) - OTP input only accepts 8
- "VT-12345" (with hyphen) - handled by .replace("-", "")
- "vt123456" (lowercase) - handled by .toUpperCase()

// Invalid formats that should fail:
- "11234567" (practitioner prefix) - correctly rejected
- "VT12345" (too short) - correctly rejected
- "VT12345678" (too long for OTP input) - truncated
```

**Root Causes:**
1. OTP input hardcoded to 8 characters but VT/BS kits need 9
2. Schema validation happens after OTP input truncation
3. No real-time format detection
4. Error messages don't explain format requirements

**Impact:**
- Customers with VT/BS kits cannot register (CRITICAL)
- Confusing error messages
- High support ticket volume
- Customer frustration

**Mitigation:**
- **URGENT**: Make OTP input dynamic (8 or 9 chars based on prefix)
- Add format hints: "VT/BS kits: 9 characters, DP/00 kits: 8 characters"
- Real-time validation feedback

---

### 2. **Date Validation Edge Cases**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 140-141

**Issue:**
```javascript
const maxDate = new Date(); // Today
const minDate = new Date(new Date().getFullYear() - 2, 0, 1); // 2 years ago
```

**Failure Scenarios:**
- Same as practitioner flow
- Timezone mismatches (client vs server)
- Boundary conditions (exactly 2 years ago)
- Date serialization issues

**Impact:**
- Valid samples rejected
- Data inconsistency

---

### 3. **Kit Already Registered Conflict**

**Risk Level:** 🔴 HIGH

**Location:** `RegisterKit.tsx` lines 114-121

**Issue:**
```javascript
if (isAlreadyRegisteredConflict(error)) {
  toastError(KIT_REGISTRATION_CONFLICT_MESSAGE);
  return;
}
```

**Failure Scenarios:**
- Kit already registered by same customer (different account?)
- Kit registered by practitioner on behalf of customer
- Kit registered in test environment
- No way to link existing kit to account

**Root Cause:**
- HTTP 409 conflict with no context
- No resolution workflow
- No "claim existing kit" feature

**Impact:**
- Customers blocked from accessing their results
- Manual support intervention required
- Poor customer experience

---

### 4. **Auto-Registration Exemption Modal**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 100-110, 178-182

**Issue:**
```javascript
if (outcome.type === "already_registered_auto") {
  setIsExemptionModalOpen(true);
  return;
}
```

**Failure Scenarios:**
- Same as practitioner flow
- Modal doesn't explain auto-registration
- No clear next steps
- Form state lost on modal close

**Impact:**
- Customer confusion
- Abandoned registrations

---

## CRITICAL RISKS - Health Information (Step 2)

### 5. **SessionStorage Data Persistence Issues**

**Risk Level:** 🔴 HIGH

**Location:** `HealthInfo.tsx` lines 205-244, `store/response.tsx`

**Issue:**
```javascript
useEffect(() => {
  if (
    !userHook.isLoading &&
    !userHook.error &&
    userData &&
    userData.kit &&
    userData.kit.status === KitStatus.REGISTERED
  ) {
    if (data && !responses.length) {
      const initialResponse: ResponseInterface[] = data.map((category) => ({
        kitId: userData.kit.kitNumber,
        categoryId: category.categoryId,
        completed: false,
        questionsResponse: [],
      }));
      setInitialResponse(initialResponse);
    } else if (responses.some((rs) => rs.kitId !== userData.kit.kitNumber)) {
      // Complex logic to merge responses from different kits
    }
  }
}, [data]);
```

**Failure Scenarios:**
- SessionStorage quota exceeded (5-10MB limit)
- SessionStorage cleared by browser/user
- Multiple tabs open - data conflicts
- Responses from different kits mixed in store
- `kitNumber` undefined - initialization fails
- Browser private mode - sessionStorage disabled

**Root Causes:**
1. No quota management
2. No conflict resolution for multi-tab scenarios
3. Complex merge logic for different kit IDs
4. No fallback if sessionStorage unavailable
5. Data not synced to backend until final submission

**Impact:**
- Data loss (CRITICAL)
- Users re-enter all answers
- Duplicate submissions
- Inconsistent state across tabs

---

### 6. **Response Store Synchronization Issues**

**Risk Level:** 🔴 HIGH

**Location:** `HealthInfo.tsx` lines 52, `functions/index.ts` lines 42-53

**Issue:**
```javascript
const disabled = setDisabledStatus(responses, userHook?.data?.kit?.kitNumber);

export function setDisabledStatus(responses: ResponseInterface[], kitId: string) {
  if (kitId && responses.length) {
    const disabled = responses.some(
      (rs) => rs.kitId === kitId && rs.completed !== true
    );
    return disabled;
  }
  return true; // Always disabled if no kitId or no responses
}
```

**Failure Scenarios:**
- `kitNumber` is undefined → button permanently disabled
- Responses array contains old kit data
- `completed` flag not set correctly
- User completes all categories but button stays disabled
- SessionStorage update doesn't trigger re-render

**Root Causes:**
1. No null safety for kitNumber
2. Responses not filtered before checking
3. `completed` flag logic unclear
4. No loading state differentiation

**Impact:**
- Submit button permanently disabled (CRITICAL)
- Users cannot proceed
- Workflow abandonment

---

### 7. **Question Submission Payload Transformation**

**Risk Level:** 🔴 HIGH

**Location:** `HealthInfo.tsx` lines 55-88

**Issue:**
```javascript
async function handleSubmit() {
  try {
    setButtonLoading(true);
    const payload = [];

    for (const rs of responses) {
      for (const qs of rs.questionsResponse) {
        const answerPayload = [];
        for (const ans of qs.answers) {
          answerPayload.push({
            ...ans,
            selectedOptions: ans.selectedOptions?.toString(), // Array to string
          });
        }
        payload.push({ ...qs, answers: answerPayload });
      }
    }

    await questionRequest.submitQuestions({ responses: payload });
    await kitRequest.updateKitStatus(userHook.data.kit.kitNumber, {
      status: KitStatus.AWAITNG_SAMPLE,
      date: new Date(),
      submitted: true,
      healthInfoCompleted: HealthInfoStatus.YES,
    });
    await kitHook.revalidate();
    setButtonLoading(false);
    setSubmitted(true);
    handleNextPage();
  } catch (error) {
    setButtonLoading(false);
    axiosErrorToast(error);
  }
}
```

**Failure Scenarios:**
- `selectedOptions` array converted to string - **DATA LOSS** for multi-select
- Empty responses array submits nothing but shows success
- `kitNumber` is undefined - API call fails
- First API call succeeds but second fails - **INCONSISTENT STATE**
- Third API call (revalidate) fails - UI not updated
- No validation of answer completeness
- No transaction rollback on partial failure

**Root Causes:**
1. Lossy data transformation (array → string)
2. No transaction handling for 3 sequential API calls
3. No payload validation before submission
4. No rollback mechanism
5. Error handling only shows toast, doesn't prevent navigation

**Impact:**
- **CRITICAL**: Multi-select answers corrupted in database
- Inconsistent state: kit marked as submitted but questions not saved
- Users think they submitted but data partially lost
- No way to recover without re-submission

**Mitigation:**
- **URGENT**: Fix selectedOptions transformation (keep as array or use proper serialization)
- Add transaction handling or backend endpoint that handles both operations
- Validate payload structure before submission
- Add optimistic UI updates with rollback

---

### 8. **Skip All Functionality Risks**

**Risk Level:** 🟡 MEDIUM

**Location:** `HealthInfo.tsx` lines 90-122

**Issue:**
```javascript
async function handleSkipAll() {
  try {
    setSkipLoading(true);
    const payload = [];

    for (const rs of responses) {
      for (const qs of rs.questionsResponse) {
        const answerPayload = [];
        for (const ans of qs.answers) {
          answerPayload.push({
            ...ans,
            selectedOptions: ans.selectedOptions?.toString(),
          });
        }
        payload.push({ ...qs, answers: answerPayload });
      }
    }

    await questionRequest.submitQuestions({ responses: payload });
    await kitRequest.updateKitStatus(userHook.data.kit.kitNumber, {
      status: KitStatus.AWAITNG_SAMPLE,
      date: new Date(),
      submitted: true,
      // NOTE: healthInfoCompleted NOT set to YES
    });
    await kitHook.revalidate();
    setSkipLoading(false);
    setSubmitted(true);
    handleNextPage();
  } catch (error) {
    setSkipLoading(false);
    axiosErrorToast(error);
  }
}
```

**Failure Scenarios:**
- User clicks "Skip all" but partial answers already entered - those are submitted
- `healthInfoCompleted` not set to YES - inconsistent with "submitted: true"
- Same data transformation issues as handleSubmit
- No confirmation dialog - accidental clicks
- Button not disabled during loading - double submission possible
- Unclear what "skip all" means (skip remaining? skip everything?)

**Root Causes:**
1. Misleading button label
2. No confirmation dialog
3. Submits partial data without user awareness
4. Inconsistent status flags
5. Same payload transformation bugs

**Impact:**
- Users accidentally skip important health information
- Partial data submitted without user knowledge
- Inconsistent database state
- Poor quality health data for analysis

---

### 9. **Category Loading and Initialization**

**Risk Level:** 🟡 MEDIUM

**Location:** `HealthInfo.tsx` lines 199-279

**Issue:**
```javascript
const Category = ({ userHook }: CategoryProps) => {
  const navigate = useNavigate();
  const { data, isLoading, error } = useCategory();
  const { responses, setInitialResponse } = useResponseStore();
  const userData = userHook.data;

  useEffect(() => {
    if (
      !userHook.isLoading &&
      !userHook.error &&
      userData &&
      userData.kit &&
      userData.kit.status === KitStatus.REGISTERED
    ) {
      if (data && !responses.length) {
        const initialResponse: ResponseInterface[] = data.map((category) => ({
          kitId: userData.kit.kitNumber,
          categoryId: category.categoryId,
          completed: false,
          questionsResponse: [],
        }));
        setInitialResponse(initialResponse);
      } else if (responses.some((rs) => rs.kitId !== userData.kit.kitNumber)) {
        // Complex merge logic with syntax error
        let initialResponse: ResponseInterface[] = [];
        const allowedResponses = responses.filter(
          (rs) => rs.kitId === userData.kit.kitNumber
        );
        initialResponse.push(...allowedResponses);
        const responseCategoryIds = allowedResponses.flatMap(
          (rs) => rs.categoryId
        );
        for (const category of data) {
          if (!responseCategoryIds.includes(category.categoryId))
            [  // SYNTAX ERROR: Extra brackets
              initialResponse.push({
                kitId: userData.kit.kitNumber,
                categoryId: category.categoryId,
                completed: false,
                questionsResponse: [],
              }),
            ];
        }
        setInitialResponse(initialResponse);
      }
    }
  }, [data]);
```

**Failure Scenarios:**
- `useCategory` fails - shows "No category available"
- `data` is empty array - shows "No category available"
- `userData.kit.kitNumber` is undefined - initialization fails
- Kit status changed to AWAITING_SAMPLE - initialization skipped
- **Syntax error in merge logic (lines 232-239)** - code may not execute as intended
- useEffect dependency array missing critical dependencies
- Multiple kits in sessionStorage - merge logic complex and error-prone

**Root Causes:**
1. No retry mechanism for failed category fetch
2. Complex conditional logic with many edge cases
3. **Syntax error in array push (extra brackets)**
4. Incomplete dependency array
5. No cache invalidation strategy
6. Kit status check too restrictive

**Impact:**
- Users cannot access questionnaire
- Initialization fails silently
- Data from wrong kit loaded
- No way to recover without page refresh

---

### 10. **Kit Status Transition Logic**

**Risk Level:** 🟡 MEDIUM

**Location:** `index.tsx` lines 60-88

**Issue:**
```javascript
useEffect(() => {
  if (
    userHook.data &&
    userHook.data.kit &&
    userHook.data.kit.status === KitStatus.REGISTERED &&
    !reset &&
    !update &&
    !userHook.data?.kit?.submitted
  ) {
    handleStep(2); // Auto-advance to health info
  } else if (
    userHook.data &&
    userHook.data.kit &&
    (userHook.data.kit.status === KitStatus.AWAITNG_SAMPLE ||
      userHook.data.kit.status === KitStatus.SAMPLE_RECEIVED ||
      userHook.data.kit.status === KitStatus.LAB_PROCESSING ||
      userHook.data.kit.status === KitStatus.RESULT_READY) &&
    !reset &&
    !update &&
    userHook.data?.kit?.submitted
  ) {
    handleStep(3); // Auto-advance to submitted
  }
}, [userHook.data]);
```

**Failure Scenarios:**
- User manually navigates back - auto-advanced forward again
- `reset` or `update` flags not set correctly
- `kit.submitted` is undefined (not false) - wrong step shown
- Race condition: `userHook.data` updates multiple times
- Long list of status checks - easy to miss one
- No validation before auto-advancing
- Infinite loop if `handleStep` triggers data reload

**Root Causes:**
1. Boolean logic doesn't handle undefined/null
2. Complex status enum checks
3. useEffect runs on every data change
4. No dependency on `reset` and `update`
5. Auto-navigation overrides user intent

**Impact:**
- Users stuck in wrong step
- Cannot edit previous steps
- Confusing navigation behavior
- Data entry errors

---

### 11. **Multi-Tab Synchronization**

**Risk Level:** 🔴 HIGH

**Location:** `store/response.tsx` (sessionStorage-based store)

**Issue:**
- SessionStorage is tab-specific (not shared across tabs)
- No synchronization between tabs
- User opens multiple tabs - different state in each

**Failure Scenarios:**
- User opens kit registration in 2 tabs
- Fills health info in tab 1
- Switches to tab 2 - sees empty form
- Fills different answers in tab 2
- Submits from tab 2 - overwrites tab 1 data
- SessionStorage in tab 1 has stale data

**Root Causes:**
1. SessionStorage is not shared across tabs
2. No tab synchronization mechanism
3. No conflict detection
4. No "last write wins" strategy

**Impact:**
- Data loss
- User confusion
- Duplicate/conflicting submissions
- Poor UX

**Mitigation:**
- Use localStorage instead (shared across tabs)
- Add storage event listeners for cross-tab sync
- Add conflict detection
- Show warning if multiple tabs detected

---

### 12. **Browser Storage Limitations**

**Risk Level:** 🟡 MEDIUM

**Location:** `store/response.tsx`

**Issue:**
- SessionStorage quota: 5-10MB (browser-dependent)
- No quota management
- Large questionnaire responses can exceed quota

**Failure Scenarios:**
- User answers many questions with long text
- SessionStorage quota exceeded
- `setItem()` throws QuotaExceededError
- Store update fails silently
- User loses all progress

**Root Causes:**
1. No quota checking
2. No error handling for storage failures
3. No data compression
4. No cleanup of old data

**Impact:**
- Data loss
- Silent failures
- Workflow abandonment

---

## ADDITIONAL RISKS

### 13. **Network and Infrastructure Failures**

**Risk Level:** 🔴 HIGH

**Common Failure Scenarios:**
- API endpoint timeouts
- CORS issues
- Authentication token expiration
- Rate limiting
- Server 500 errors with no retry
- Network disconnection during submission

**Impact:**
- Complete workflow failure
- Data loss
- User frustration

---

### 14. **Form State Management**

**Risk Level:** 🟡 MEDIUM

**Location:** `RegisterKit.tsx` lines 18-52

**Issue:**
- Custom form hook manages state
- No form state persistence
- Page refresh loses all data
- No "save draft" functionality

**Impact:**
- Data loss on accidental refresh
- Poor UX for long forms

---

### 15. **Browser Compatibility**

**Risk Level:** 🟡 MEDIUM

**Potential Issues:**
- DatePicker component rendering issues
- OTP input focus behavior inconsistent
- SessionStorage disabled in private mode
- SessionStorage quota varies by browser
- Browser back button breaks state

**Impact:**
- Feature unavailable for some users
- Inconsistent UX across browsers

---

## COMPARISON: Customer vs Practitioner Flow

| Aspect | Customer Flow | Practitioner Flow | Risk Difference |
|--------|---------------|-------------------|-----------------|
| **State Management** | SessionStorage (tab-specific) | SSE (real-time, server-synced) | Customer: Multi-tab conflicts; Practitioner: Connection failures |
| **Kit ID Validation** | Stricter (VT/BS/DP/00 only) | More permissive (includes 11-99) | Customer: VT/BS kits fail due to 8-char OTP limit |
| **Client Name** | Not required | Required | Practitioner: Additional validation point |
| **Skip Functionality** | Available | Not available | Customer: Risk of accidental skips |
| **Data Persistence** | Lost on tab close | Synced to server | Customer: Higher data loss risk |
| **Error Recovery** | Page refresh required | SSE reconnection possible | Customer: Worse recovery UX |

---

## RECOMMENDATIONS

### Immediate Actions (CRITICAL Priority)

1. **Fix OTP Input Length for VT/BS Kits**
   - **URGENT**: Make OTP input dynamic (8 or 9 chars)
   - Add format detection based on first 2 characters
   - This is blocking customers from registering

2. **Fix selectedOptions Data Loss**
   - **URGENT**: Remove `.toString()` conversion
   - Keep as array or use proper JSON serialization
   - This is corrupting multi-select answers in database

3. **Fix Submit Button Disabled State**
   - Add null safety for kitNumber
   - Filter responses by kitId before checking completion
   - Add loading state differentiation

4. **Add Transaction Handling**
   - Wrap multiple API calls in try-catch
   - Implement rollback on partial failure
   - Or create single backend endpoint for atomic operation

### High Priority

5. **Improve SessionStorage Reliability**
   - Switch to localStorage for cross-tab sync
   - Add storage event listeners
   - Add quota management
   - Add error handling for QuotaExceededError

6. **Fix Category Initialization**
   - Fix syntax error in merge logic (lines 232-239)
   - Add retry mechanism for category fetch
   - Improve error states

7. **Add Skip Confirmation**
   - Add confirmation dialog for "Skip all"
   - Clarify what data will be submitted
   - Disable button during loading

### Medium Priority

8. **Enhance Error Recovery**
   - Add retry buttons on error states
   - Preserve form state on navigation
   - Add error telemetry

9. **Improve Auto-Navigation**
   - Add explicit user confirmation
   - Fix boolean logic for undefined/null
   - Prevent infinite loops

10. **Add Form State Persistence**
    - Save draft to localStorage
    - Auto-save on field blur
    - Restore on page load

### Long-term Improvements

11. **Add Monitoring**
    - Track registration success rate by kit type
    - Monitor sessionStorage failures
    - Track multi-tab conflicts
    - Alert on high error rates

12. **Improve UX**
    - Add progress indicators
    - Better loading states
    - Clearer error messages
    - Multi-tab warning

13. **Code Quality**
    - Fix syntax errors
    - Add TypeScript strict mode
    - Add comprehensive unit tests
    - Add integration tests

---

## MONITORING SUGGESTIONS

### Key Metrics to Track

1. **Kit Registration Success Rate**
   - Track by kit prefix (VT/BS/DP/00)
   - Monitor validation failures
   - Track 409 conflicts
   - **Alert if VT/BS registration rate < 50%** (indicates OTP length issue)

2. **Health Info Completion Rate**
   - SessionStorage initialization success
   - Category fetch success rate
   - Submission success rate
   - Skip rate (should be low)
   - Time to complete questionnaire

3. **Error Rates**
   - API errors by endpoint
   - SessionStorage quota errors
   - Client-side errors by component
   - Network failures
   - Multi-tab conflicts

4. **User Behavior**
   - Step abandonment rates
   - Back button usage
   - Form field errors
   - Support ticket correlation
   - Multi-tab usage patterns

5. **Data Quality**
   - Incomplete submissions
   - Skipped health info rate
   - Multi-select answer corruption
   - Partial submission failures

---

## CRITICAL BUGS TO FIX IMMEDIATELY

1. **OTP Input Length** - Blocking VT/BS kit registration
2. **selectedOptions Data Loss** - Corrupting database
3. **Submit Button Disabled** - Blocking workflow completion
4. **Syntax Error in Category Merge** - Potential runtime error

---

**Document Version:** 1.0
**Last Updated:** 2026-02-23
**Next Review:** Quarterly or after major changes

