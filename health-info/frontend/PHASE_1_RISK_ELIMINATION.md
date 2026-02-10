# Phase 1: Risk Elimination

**Status:** MANDATORY  
**Priority:** CRITICAL  
**Blocks:** All other phases

---

## Goal

Eliminate all data corruption, state leakage, and broken user flows that can:
- Cause users to submit incorrect health information
- Lose or corrupt user data
- Break on refresh, navigation, or back button
- Leak data between kits
- Silently fail without user awareness

---

## Why This Phase Exists

The current implementation has **critical safety issues** that make the application unreliable:

1. **Cross-kit data leakage:** Users can see/submit another kit's health data
2. **Broken refresh flows:** Pages crash or show empty state after refresh
3. **Dual-store inconsistencies:** Two sources of truth conflict
4. **Missing edit enforcement:** Submitted kits can be edited without validation
5. **Unsafe cache behavior:** Stale data persists across kit switches

These issues **must be fixed before any other work** because:
- UX improvements built on broken state are wasted effort
- Refactoring stores while leakage exists compounds the problem
- Adding features to corrupt flows increases risk

---

## Included Issues (Mapped to Audit Findings)

From `assessments/FRONTEND_AUDIT_REPORT.md`:

- **Issue 1:** Cross-kit data leakage in Zustand stores
- **Issue 2:** Broken refresh/navigation (location.state dependency)
- **Issue 3:** Dual-store inconsistencies (Zustand + React Query)
- **Issue 4:** Missing edit enforcement (submitted status not checked)
- **Issue 5:** Unsafe cache behavior (stale data across kit switches)
- **Issue 6:** Incorrect data source usage (external API vs internal)
- **Issue 7:** State corruption on navigation

---

## Tasks

### Task 1.1: Fix Cross-Kit Data Leakage in Zustand Stores

**Description:**  
The `useHealthInfoStore` and `useHealthInfoFormStore` are global singletons that don't isolate state by kit ID. When switching between kits, previous kit's data persists.

**Why Phase 1:**  
This can cause users to **submit Kit A's health data to Kit B**, which is a critical data integrity violation.

**Files Affected:**
- `src/stores/healthInfoStore.ts`
- `src/stores/healthInfoFormStore.ts`
- All components using these stores

**Implementation:**

1. **Add kit ID scoping to stores:**
   ```typescript
   // Before: Global state
   answers: Record<string, any>
   
   // After: Kit-scoped state
   answersByKit: Record<string, Record<string, any>>
   currentKitId: string | null
   ```

2. **Add store reset on kit switch:**
   ```typescript
   setCurrentKit: (kitId: string) => {
     set({ currentKitId: kitId })
     // Clear previous kit's data from memory
   }
   ```

3. **Add selectors that enforce kit isolation:**
   ```typescript
   getAnswersForCurrentKit: () => {
     const { answersByKit, currentKitId } = get()
     return currentKitId ? answersByKit[currentKitId] : {}
   }
   ```

**Acceptance Criteria:**
- [ ] Switching between Kit A and Kit B shows correct data for each
- [ ] No answers from Kit A appear when viewing Kit B
- [ ] Store state is cleared when navigating away from health info
- [ ] Unit tests verify kit isolation

**Rollout Notes:**
- Test with multiple kits in different states (draft, submitted)
- Verify no data leakage in browser DevTools (Zustand state)
- Test rapid kit switching

---

### Task 1.2: Fix Broken Refresh/Navigation Flows

**Description:**  
Pages like `HealthInfoStep2.tsx` rely on `location.state` to get kit ID and health info ID. This breaks on refresh, direct links, and back button navigation.

**Why Phase 1:**  
Users **cannot complete the health info flow** if they refresh or use the back button. This breaks a core user journey.

**Files Affected:**
- `src/pages/HealthInfoStep2.tsx`
- `src/pages/HealthInfoStep3.tsx`
- `src/pages/EditHealthInfo.tsx`
- Any component using `location.state` for critical data

**Implementation:**

1. **Extract IDs from URL params instead of location.state:**
   ```typescript
   // Before:
   const { kitId, healthInfoId } = location.state || {}
   
   // After:
   const { kitId, healthInfoId } = useParams<{ kitId: string; healthInfoId: string }>()
   ```

2. **Update routes to include IDs:**
   ```typescript
   // Before:
   /health-info/step2
   
   // After:
   /health-info/:kitId/:healthInfoId/step2
   ```

3. **Fetch data from URL params:**
   ```typescript
   const { data: healthInfo } = useQuery({
     queryKey: ['healthInfo', healthInfoId],
     queryFn: () => fetchHealthInfo(healthInfoId),
     enabled: !!healthInfoId
   })
   ```

4. **Add loading and error states:**
   ```typescript
   if (isLoading) return <LoadingSpinner />
   if (error) return <ErrorState />
   if (!healthInfo) return <NotFound />
   ```

**Acceptance Criteria:**
- [ ] All health info pages work correctly after refresh
- [ ] Direct links to health info pages work
- [ ] Back button navigation works predictably
- [ ] No reliance on location.state for critical data
- [ ] Proper loading/error states shown

**Rollout Notes:**
- Test all navigation paths (forward, back, refresh, direct link)
- Verify URL structure is clean and bookmarkable
- Ensure no 404s or blank pages

---

### Task 1.3: Fix Dual-Store Inconsistencies

**Description:**
Health info data exists in both Zustand stores (`useHealthInfoStore`, `useHealthInfoFormStore`) and React Query cache. These can become out of sync, causing stale data to be displayed or submitted.

**Why Phase 1:**
Dual sources of truth can cause users to **see outdated data** or **submit stale answers**, leading to data corruption.

**Files Affected:**
- `src/stores/healthInfoStore.ts`
- `src/stores/healthInfoFormStore.ts`
- `src/hooks/useHealthInfo.ts`
- All components reading health info data

**Implementation:**

1. **Establish React Query as single source of truth:**
   - Use React Query for all server state (health info, answers)
   - Use Zustand only for UI state (current step, form dirty state)

2. **Remove duplicate data from Zustand:**
   ```typescript
   // Remove from Zustand:
   - answers
   - healthInfo
   - sections

   // Keep in Zustand:
   - currentStep (UI state)
   - isDirty (UI state)
   - validationErrors (UI state)
   ```

3. **Update components to read from React Query:**
   ```typescript
   // Before:
   const answers = useHealthInfoStore(state => state.answers)

   // After:
   const { data: answers } = useQuery({
     queryKey: ['healthInfo', healthInfoId, 'answers'],
     queryFn: () => fetchAnswers(healthInfoId)
   })
   ```

4. **Invalidate cache on mutations:**
   ```typescript
   const mutation = useMutation({
     mutationFn: saveAnswers,
     onSuccess: () => {
       queryClient.invalidateQueries(['healthInfo', healthInfoId])
     }
   })
   ```

**Acceptance Criteria:**
- [ ] React Query is the only source for server data
- [ ] Zustand only contains UI state
- [ ] No stale data shown after mutations
- [ ] Cache invalidation works correctly
- [ ] No duplicate data storage

**Rollout Notes:**
- Audit all components for Zustand usage
- Verify cache invalidation triggers correctly
- Test concurrent edits (if applicable)

---

### Task 1.4: Fix Missing Edit Enforcement (Submitted Status)

**Description:**
The edit flow doesn't check if a health info record is already submitted. Users can edit submitted records without proper validation or warnings.

**Why Phase 1:**
Editing submitted health info can **corrupt finalized data** and **violate business rules** (e.g., practitioner review invalidation).

**Files Affected:**
- `src/pages/EditHealthInfo.tsx`
- `src/components/HealthInfoForm.tsx`
- Edit mutation hooks

**Implementation:**

1. **Add submitted status check on edit page load:**
   ```typescript
   const { data: healthInfo } = useQuery(['healthInfo', healthInfoId])

   useEffect(() => {
     if (healthInfo?.status === 'submitted') {
       // Show warning modal or redirect
       setShowSubmittedWarning(true)
     }
   }, [healthInfo])
   ```

2. **Add backend validation:**
   ```typescript
   // Ensure backend rejects edits to submitted records
   // OR requires explicit "reopen" action
   ```

3. **Add UI indicators:**
   ```typescript
   {healthInfo?.status === 'submitted' && (
     <Alert severity="warning">
       This health information has been submitted.
       Editing will require re-submission and practitioner re-review.
     </Alert>
   )}
   ```

4. **Add confirmation flow:**
   ```typescript
   const handleEdit = () => {
     if (healthInfo?.status === 'submitted') {
       showConfirmDialog({
         title: 'Edit Submitted Health Info?',
         message: 'This will reopen the record for editing...',
         onConfirm: () => reopenAndEdit()
       })
     }
   }
   ```

**Acceptance Criteria:**
- [ ] Cannot edit submitted health info without explicit confirmation
- [ ] Clear warning shown when editing submitted records
- [ ] Backend validates submitted status
- [ ] Status changes are tracked (submitted → draft)
- [ ] Practitioner is notified of re-edits (if applicable)

**Rollout Notes:**
- Test with submitted records
- Verify backend validation
- Ensure status transitions are logged

---

### Task 1.5: Fix Unsafe Cache Behavior

**Description:**
React Query cache persists across kit switches, causing stale data to appear when switching between kits quickly.

**Why Phase 1:**
Stale cache can show **Kit A's data when viewing Kit B**, causing data leakage and user confusion.

**Files Affected:**
- `src/hooks/useHealthInfo.ts`
- `src/providers/QueryProvider.tsx`
- All React Query configurations

**Implementation:**

1. **Add kit ID to all query keys:**
   ```typescript
   // Before:
   queryKey: ['healthInfo', healthInfoId]

   // After:
   queryKey: ['healthInfo', kitId, healthInfoId]
   ```

2. **Invalidate cache on kit switch:**
   ```typescript
   const handleKitSwitch = (newKitId: string) => {
     // Clear cache for previous kit
     queryClient.removeQueries(['healthInfo', previousKitId])
     setCurrentKitId(newKitId)
   }
   ```

3. **Add staleTime configuration:**
   ```typescript
   const queryClient = new QueryClient({
     defaultOptions: {
       queries: {
         staleTime: 30000, // 30 seconds
         cacheTime: 300000, // 5 minutes
         refetchOnWindowFocus: true
       }
     }
   })
   ```

4. **Add cache boundaries:**
   ```typescript
   // Clear all health info cache when leaving health info section
   useEffect(() => {
     return () => {
       queryClient.removeQueries(['healthInfo'])
     }
   }, [])
   ```

**Acceptance Criteria:**
- [ ] No stale data shown when switching kits
- [ ] Cache is properly scoped by kit ID
- [ ] Cache is cleared when leaving health info section
- [ ] Refetch behavior is predictable
- [ ] No memory leaks from unbounded cache

**Rollout Notes:**
- Test rapid kit switching
- Monitor cache size in DevTools
- Verify refetch behavior

---

### Task 1.6: Fix Incorrect Data Source Usage

**Description:**
Some components fetch from external API (`/api/external/health-info`) while others use internal API (`/api/health-info`). This causes inconsistencies and potential data mismatches.

**Why Phase 1:**
Using wrong data source can show **incorrect or outdated data**, breaking user trust and data integrity.

**Files Affected:**
- `src/services/healthInfoService.ts`
- `src/hooks/useHealthInfo.ts`
- All components fetching health info

**Implementation:**

1. **Audit all API calls:**
   ```bash
   # Find all external API usage
   grep -r "/api/external/health-info" src/
   ```

2. **Standardize on internal API:**
   ```typescript
   // Replace all external API calls with internal
   // Before:
   fetch('/api/external/health-info')

   // After:
   fetch('/api/health-info')
   ```

3. **Add migration plan for external API:**
   ```typescript
   // If external API is still needed:
   // 1. Document why
   // 2. Add clear separation (e.g., useExternalHealthInfo hook)
   // 3. Add comments explaining usage
   ```

4. **Add API source validation:**
   ```typescript
   // Add runtime checks to ensure correct API is used
   const validateApiSource = (url: string) => {
     if (url.includes('/external/') && !isExternalAllowed()) {
       console.error('Unexpected external API usage:', url)
     }
   }
   ```

**Acceptance Criteria:**
- [ ] All health info fetches use consistent API
- [ ] No unexpected external API calls
- [ ] Clear documentation of API usage
- [ ] Migration plan for external API (if needed)
- [ ] No data mismatches between sources

**Rollout Notes:**
- Coordinate with backend team
- Test all data fetch scenarios
- Verify no regressions

---

### Task 1.7: Fix State Corruption on Navigation

**Description:**
Navigating between health info pages can leave stores in inconsistent states (e.g., answers from previous page persist).

**Why Phase 1:**
State corruption can cause **wrong answers to be submitted** or **UI to show incorrect data**.

**Files Affected:**
- All health info page components
- `src/stores/healthInfoStore.ts`
- Navigation hooks

**Implementation:**

1. **Add cleanup on unmount:**
   ```typescript
   useEffect(() => {
     return () => {
       // Clear form state when leaving page
       healthInfoFormStore.reset()
     }
   }, [])
   ```

2. **Add navigation guards:**
   ```typescript
   const handleNavigate = (to: string) => {
     if (isDirty) {
       showConfirmDialog({
         message: 'You have unsaved changes. Continue?',
         onConfirm: () => navigate(to)
       })
     } else {
       navigate(to)
     }
   }
   ```

3. **Add state validation on page load:**
   ```typescript
   useEffect(() => {
     const isStateValid = validateHealthInfoState()
     if (!isStateValid) {
       // Reset to safe state
       healthInfoStore.reset()
       // Refetch from server
       refetch()
     }
   }, [])
   ```

4. **Add store reset on route change:**
   ```typescript
   useEffect(() => {
     // Reset stores when route changes
     return () => {
       if (!isHealthInfoRoute(location.pathname)) {
         healthInfoStore.reset()
         healthInfoFormStore.reset()
       }
     }
   }, [location.pathname])
   ```

**Acceptance Criteria:**
- [ ] No stale state persists across navigation
- [ ] Stores are reset when leaving health info section
- [ ] Navigation guards prevent data loss
- [ ] State is validated on page load
- [ ] No orphaned state in stores

**Rollout Notes:**
- Test all navigation paths
- Verify cleanup happens correctly
- Test rapid navigation

---

## Exit Criteria

Phase 1 is complete when **ALL** of the following are true:

### Data Integrity
- [ ] No cross-kit data leakage (verified with multiple kits)
- [ ] No stale data shown after mutations
- [ ] No data corruption on navigation
- [ ] Submitted status is enforced

### Navigation & Refresh
- [ ] All pages work correctly after refresh
- [ ] Direct links work for all health info pages
- [ ] Back button works predictably
- [ ] No reliance on location.state for critical data

### Store & Cache
- [ ] Single source of truth for server data (React Query)
- [ ] Zustand only contains UI state
- [ ] Cache is properly scoped by kit ID
- [ ] Cache invalidation works correctly

### API Consistency
- [ ] All health info fetches use consistent API
- [ ] No unexpected external API calls
- [ ] Clear documentation of API usage

### Testing
- [ ] Unit tests for kit isolation
- [ ] Integration tests for navigation flows
- [ ] Manual testing with multiple kits
- [ ] No console errors or warnings

### Rollout Safety
- [ ] Gradual rollout plan defined
- [ ] Rollback plan documented
- [ ] Monitoring in place
- [ ] User acceptance testing complete

---

## Risk Reduction Estimate

Completing Phase 1 eliminates approximately **90% of critical bugs**:

- Cross-kit leakage: **ELIMINATED**
- Refresh failures: **ELIMINATED**
- Dual-store conflicts: **ELIMINATED**
- Edit enforcement: **ENFORCED**
- Cache staleness: **ELIMINATED**
- API inconsistencies: **RESOLVED**
- Navigation corruption: **ELIMINATED**

---

## Next Steps

1. Review all tasks with engineering team
2. Estimate effort for each task
3. Assign owners
4. Begin Task 1.1 (highest priority)
5. Complete all tasks sequentially
6. Verify exit criteria
7. **Only then** proceed to Phase 2

**Do NOT start Phase 2 until all Phase 1 exit criteria are met.**

