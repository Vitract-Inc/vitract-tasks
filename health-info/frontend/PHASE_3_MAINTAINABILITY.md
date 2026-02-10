# Phase 3: Maintainability

**Status:** REQUIRED  
**Priority:** MEDIUM  
**Depends On:** Phase 1 and Phase 2 complete

---

## Goal

Reduce technical debt and improve code quality to:
- Unify dual Zustand stores into single source of truth
- Establish clear state boundaries
- Reduce component duplication
- Improve testability
- Document state management patterns

---

## Why This Phase Exists

After Phase 1 eliminates data corruption and Phase 2 standardizes UX, Phase 3 addresses **technical debt** that:
- Makes future changes harder
- Increases bug surface area
- Slows down development
- Makes onboarding difficult

These issues don't affect users directly, but they:
- Increase maintenance cost
- Make bugs more likely
- Slow down feature development
- Reduce code quality

**Phase 3 can only begin after Phases 1 and 2** because:
- Refactoring broken state is dangerous
- Cleaning up code before UX is stable wastes effort
- Store unification requires stable data flows

---

## Included Issues (Mapped to Audit Findings)

From `assessments/FRONTEND_AUDIT_REPORT.md`:

- **Issue 13:** Dual Zustand stores (useHealthInfoStore, useHealthInfoFormStore)
- **Issue 14:** Unclear state boundaries
- **Issue 15:** Component duplication
- **Issue 16:** Poor testability
- **Issue 17:** Lack of documentation

---

## Tasks

### Task 3.1: Unify Dual Zustand Stores

**Description:**  
Currently there are two Zustand stores for health info (`useHealthInfoStore`, `useHealthInfoFormStore`). This creates confusion about which store to use and increases maintenance burden.

**Why Phase 3:**  
This is a **code quality issue**, not a user-facing bug. After Phase 1 fixes data leakage, we can safely unify stores.

**Files Affected:**
- `src/stores/healthInfoStore.ts`
- `src/stores/healthInfoFormStore.ts`
- All components using these stores

**Implementation:**

1. **Analyze current store usage:**
   ```bash
   # Find all usages
   grep -r "useHealthInfoStore" src/
   grep -r "useHealthInfoFormStore" src/
   ```

2. **Design unified store structure:**
   ```typescript
   // src/stores/healthInfoStore.ts
   interface HealthInfoStore {
     // UI State (from form store)
     currentStep: number
     isDirty: boolean
     validationErrors: Record<string, string>
     
     // Kit scoping (from Phase 1)
     currentKitId: string | null
     
     // Actions
     setCurrentStep: (step: number) => void
     setDirty: (dirty: boolean) => void
     setValidationErrors: (errors: Record<string, string>) => void
     setCurrentKit: (kitId: string) => void
     reset: () => void
   }
   ```

3. **Migrate components to unified store:**
   ```typescript
   // Before:
   const currentStep = useHealthInfoFormStore(state => state.currentStep)
   const kitId = useHealthInfoStore(state => state.kitId)
   
   // After:
   const { currentStep, currentKitId } = useHealthInfoStore()
   ```

4. **Remove old store:**
   ```bash
   # After all migrations complete
   rm src/stores/healthInfoFormStore.ts
   ```

5. **Add store documentation:**
   ```typescript
   /**
    * Health Info Store
    * 
    * Manages UI state for health information forms.
    * 
    * NOTE: This store only contains UI state.
    * Server state (health info data, answers) is managed by React Query.
    * 
    * State:
    * - currentStep: Current step in multi-step form
    * - isDirty: Whether form has unsaved changes
    * - validationErrors: Form validation errors
    * - currentKitId: Currently active kit ID
    */
   ```

**Acceptance Criteria:**
- [ ] Single Zustand store for health info UI state
- [ ] All components migrated to unified store
- [ ] Old store deleted
- [ ] Store is documented
- [ ] No regressions in functionality

**Rollout Notes:**
- Migrate components incrementally
- Test after each migration
- Verify no store conflicts

---

### Task 3.2: Establish Clear State Boundaries

**Description:**  
State is scattered across Zustand stores, React Query cache, component state, and URL params. Clear boundaries need to be established.

**Why Phase 3:**  
This is an **architectural clarity issue**. Clear boundaries prevent future bugs but don't fix existing user-facing issues.

**Files Affected:**
- All state management code
- Documentation

**Implementation:**

1. **Document state boundaries:**
   ```typescript
   /**
    * State Management Architecture
    * 
    * 1. Server State (React Query)
    *    - Health info records
    *    - Answers
    *    - Kit data
    *    - User data
    * 
    * 2. UI State (Zustand)
    *    - Current step
    *    - Form dirty state
    *    - Validation errors
    *    - Current kit ID
    * 
    * 3. URL State (React Router)
    *    - Kit ID
    *    - Health info ID
    *    - Current page
    * 
    * 4. Component State (useState)
    *    - Modal open/closed
    *    - Dropdown expanded
    *    - Temporary UI state
    */
   ```

2. **Add state validation:**
   ```typescript
   // Validate state is in correct location
   const validateStateLocation = () => {
     // Server data should not be in Zustand
     if (healthInfoStore.getState().answers) {
       console.error('Server data found in Zustand store')
     }
     
     // UI state should not be in React Query
     if (queryClient.getQueryData(['currentStep'])) {
       console.error('UI state found in React Query')
     }
   }
   ```

3. **Add linting rules:**
   ```typescript
   // ESLint rule to prevent server data in Zustand
   // ESLint rule to prevent UI state in React Query
   ```

4. **Create state management guide:**
   ```markdown
   # State Management Guide
   
   ## When to use React Query
   - Data from API
   - Data that needs to be cached
   - Data that can be stale
   
   ## When to use Zustand
   - UI state that needs to be shared
   - State that doesn't come from API
   - State that needs to persist across unmounts
   
   ## When to use URL params
   - State that should be bookmarkable
   - State that should survive refresh
   - Navigation state
   
   ## When to use component state
   - Temporary UI state
   - State that doesn't need to be shared
   - State that should reset on unmount
   ```

**Acceptance Criteria:**
- [ ] State boundaries documented
- [ ] State validation in place
- [ ] Linting rules added
- [ ] State management guide created
- [ ] Team trained on boundaries

**Rollout Notes:**
- Review with team
- Add to onboarding docs
- Enforce in code reviews

---

### Task 3.3: Reduce Component Duplication

**Description:**
Similar components exist for practitioner and client flows, and across different health info pages. This increases maintenance burden.

**Why Phase 3:**
This is a **code quality issue**. Duplication doesn't affect users but makes changes harder and increases bug risk.

**Files Affected:**
- Practitioner-specific components
- Client-specific components
- Duplicated form components
- Duplicated display components

**Implementation:**

1. **Audit component duplication:**
   ```bash
   # Find similar components
   find src/components -name "*HealthInfo*" -type f
   find src/pages -name "*HealthInfo*" -type f
   ```

2. **Extract shared components:**
   ```typescript
   // Before: PractitionerHealthInfoForm, ClientHealthInfoForm

   // After: Unified component with role prop
   interface HealthInfoFormProps {
     healthInfo: HealthInfo
     role: 'practitioner' | 'client'
     readOnly?: boolean
     onSave?: (data: HealthInfo) => void
   }

   const HealthInfoForm: React.FC<HealthInfoFormProps> = ({
     healthInfo,
     role,
     readOnly,
     onSave
   }) => {
     const canEdit = !readOnly && (
       role === 'practitioner' ||
       healthInfo.status === 'draft'
     )

     return (
       <Form>
         {/* Unified form logic */}
       </Form>
     )
   }
   ```

3. **Create composition patterns:**
   ```typescript
   // Extract reusable pieces
   const HealthInfoSection = ({ section, answers, onChange }) => { ... }
   const HealthInfoQuestion = ({ question, answer, onChange }) => { ... }
   const HealthInfoAnswer = ({ answer, type }) => { ... }

   // Compose into larger components
   const HealthInfoForm = () => (
     <>
       {sections.map(section => (
         <HealthInfoSection key={section.id} section={section} />
       ))}
     </>
   )
   ```

4. **Remove duplicate components:**
   ```bash
   # After migration
   rm src/components/PractitionerHealthInfoForm.tsx
   rm src/components/ClientHealthInfoForm.tsx
   ```

5. **Document component patterns:**
   ```markdown
   # Health Info Component Architecture

   ## Core Components
   - HealthInfoForm: Main form component
   - HealthInfoSection: Section container
   - HealthInfoQuestion: Individual question
   - HealthInfoAnswer: Answer display

   ## Composition
   - Use role prop for role-specific behavior
   - Use readOnly prop for view mode
   - Use composition over duplication
   ```

**Acceptance Criteria:**
- [ ] No duplicate components for roles
- [ ] Shared components extracted
- [ ] Composition patterns documented
- [ ] Old duplicate components deleted
- [ ] No regressions in functionality

**Rollout Notes:**
- Migrate incrementally
- Test both roles after each change
- Verify no visual regressions

---

### Task 3.4: Improve Testability

**Description:**
Current components are hard to test due to tight coupling with stores, API calls, and navigation. Improve testability through better separation of concerns.

**Why Phase 3:**
This is a **code quality issue**. Better testability prevents future bugs but doesn't fix existing user-facing issues.

**Files Affected:**
- All health info components
- Test files
- Test utilities

**Implementation:**

1. **Extract business logic from components:**
   ```typescript
   // Before: Logic in component
   const HealthInfoForm = () => {
     const [answers, setAnswers] = useState({})

     const handleSave = async () => {
       const validated = validateAnswers(answers)
       if (validated.isValid) {
         await saveAnswers(answers)
       }
     }

     return <Form onSubmit={handleSave} />
   }

   // After: Logic in hook
   const useHealthInfoForm = (healthInfoId: string) => {
     const [answers, setAnswers] = useState({})

     const handleSave = async () => {
       const validated = validateAnswers(answers)
       if (validated.isValid) {
         await saveAnswers(answers)
       }
     }

     return { answers, setAnswers, handleSave }
   }

   const HealthInfoForm = () => {
     const { answers, setAnswers, handleSave } = useHealthInfoForm(healthInfoId)
     return <Form onSubmit={handleSave} />
   }
   ```

2. **Add dependency injection:**
   ```typescript
   // Allow injecting dependencies for testing
   interface HealthInfoFormProps {
     healthInfoId: string
     onSave?: (data: HealthInfo) => Promise<void>
     onValidate?: (data: HealthInfo) => ValidationResult
   }

   const HealthInfoForm: React.FC<HealthInfoFormProps> = ({
     healthInfoId,
     onSave = defaultSave,
     onValidate = defaultValidate
   }) => {
     // Use injected dependencies
   }
   ```

3. **Create test utilities:**
   ```typescript
   // src/test/healthInfoTestUtils.ts
   export const createMockHealthInfo = (overrides?: Partial<HealthInfo>) => ({
     id: 'test-id',
     kitId: 'test-kit',
     status: 'draft',
     answers: {},
     ...overrides
   })

   export const renderHealthInfoForm = (props?: Partial<HealthInfoFormProps>) => {
     return render(
       <QueryClientProvider client={testQueryClient}>
         <HealthInfoForm
           healthInfoId="test-id"
           {...props}
         />
       </QueryClientProvider>
     )
   }
   ```

4. **Add unit tests:**
   ```typescript
   // src/components/HealthInfoForm.test.tsx
   describe('HealthInfoForm', () => {
     it('renders questions correctly', () => {
       const { getByText } = renderHealthInfoForm()
       expect(getByText('Question 1')).toBeInTheDocument()
     })

     it('validates answers before save', async () => {
       const onValidate = jest.fn()
       const { getByRole } = renderHealthInfoForm({ onValidate })

       fireEvent.click(getByRole('button', { name: 'Save' }))

       expect(onValidate).toHaveBeenCalled()
     })
   })
   ```

5. **Add integration tests:**
   ```typescript
   // src/pages/EditHealthInfo.integration.test.tsx
   describe('EditHealthInfo integration', () => {
     it('saves and navigates on submit', async () => {
       const { getByRole } = render(<EditHealthInfo />)

       // Fill form
       fireEvent.change(getByRole('textbox'), { target: { value: 'Answer' } })

       // Submit
       fireEvent.click(getByRole('button', { name: 'Save' }))

       // Verify navigation
       await waitFor(() => {
         expect(mockNavigate).toHaveBeenCalledWith('/kits')
       })
     })
   })
   ```

**Acceptance Criteria:**
- [ ] Business logic extracted from components
- [ ] Dependency injection in place
- [ ] Test utilities created
- [ ] Unit tests added for core components
- [ ] Integration tests added for key flows
- [ ] Test coverage > 70%

**Rollout Notes:**
- Add tests incrementally
- Run tests in CI/CD
- Enforce test coverage in PRs

---

### Task 3.5: Document State Management Patterns

**Description:**
State management patterns are undocumented, making it hard for new developers to understand the architecture.

**Why Phase 3:**
This is a **documentation issue**. Good docs prevent future bugs but don't fix existing issues.

**Files Affected:**
- Documentation files
- Code comments
- README files

**Implementation:**

1. **Create state management documentation:**
   ```markdown
   # State Management Architecture

   ## Overview

   Health info state is managed using a combination of:
   - React Query for server state
   - Zustand for UI state
   - React Router for URL state
   - Component state for temporary UI

   ## React Query (Server State)

   ### What goes in React Query?
   - Health info records from API
   - Answers from API
   - Kit data from API
   - Any data that comes from the server

   ### Query Keys
   ```typescript
   ['healthInfo', kitId, healthInfoId]
   ['healthInfo', kitId, healthInfoId, 'answers']
   ```

   ### Cache Configuration
   - staleTime: 30 seconds
   - cacheTime: 5 minutes
   - refetchOnWindowFocus: true

   ## Zustand (UI State)

   ### What goes in Zustand?
   - Current step in multi-step form
   - Form dirty state
   - Validation errors
   - Current kit ID

   ### Store Structure
   ```typescript
   interface HealthInfoStore {
     currentStep: number
     isDirty: boolean
     validationErrors: Record<string, string>
     currentKitId: string | null
   }
   ```

   ## Common Patterns

   ### Fetching Health Info
   ```typescript
   const { data: healthInfo } = useQuery({
     queryKey: ['healthInfo', kitId, healthInfoId],
     queryFn: () => fetchHealthInfo(healthInfoId)
   })
   ```

   ### Saving Answers
   ```typescript
   const mutation = useMutation({
     mutationFn: saveAnswers,
     onSuccess: () => {
       queryClient.invalidateQueries(['healthInfo', kitId, healthInfoId])
     }
   })
   ```

   ### Managing Form State
   ```typescript
   const { currentStep, setCurrentStep } = useHealthInfoStore()
   ```
   ```

2. **Add inline documentation:**
   ```typescript
   /**
    * Health Info Store
    *
    * Manages UI state for health information forms.
    *
    * @example
    * ```typescript
    * const { currentStep, setCurrentStep } = useHealthInfoStore()
    * setCurrentStep(2)
    * ```
    */
   export const useHealthInfoStore = create<HealthInfoStore>((set) => ({
     // ...
   }))
   ```

3. **Create decision trees:**
   ```markdown
   # State Management Decision Tree

   ## Where should I put this state?

   1. Does it come from the API?
      - YES → React Query
      - NO → Continue

   2. Does it need to survive page refresh?
      - YES → URL params
      - NO → Continue

   3. Does it need to be shared across components?
      - YES → Zustand
      - NO → Component state
   ```

4. **Add examples:**
   ```markdown
   # Common Scenarios

   ## Scenario 1: Adding a new question type
   1. Add question to backend
   2. Update TypeScript types
   3. Add question component
   4. Add validation logic
   5. Add tests

   ## Scenario 2: Adding a new step
   1. Update Zustand store (add step number)
   2. Create step component
   3. Add route
   4. Update navigation
   5. Add tests
   ```

**Acceptance Criteria:**
- [ ] State management architecture documented
- [ ] Common patterns documented
- [ ] Decision trees created
- [ ] Examples added
- [ ] Inline documentation added
- [ ] Team trained on patterns

**Rollout Notes:**
- Review with team
- Add to onboarding
- Keep docs up to date

---

## Exit Criteria

Phase 3 is complete when **ALL** of the following are true:

### Store Unification
- [ ] Single Zustand store for health info UI state
- [ ] All components migrated
- [ ] Old stores deleted
- [ ] Store is documented

### State Boundaries
- [ ] State boundaries documented
- [ ] State validation in place
- [ ] Linting rules added
- [ ] Team trained

### Component Quality
- [ ] No duplicate components
- [ ] Shared components extracted
- [ ] Composition patterns documented
- [ ] Old duplicates deleted

### Testability
- [ ] Business logic extracted
- [ ] Dependency injection in place
- [ ] Test utilities created
- [ ] Unit tests added
- [ ] Integration tests added
- [ ] Test coverage > 70%

### Documentation
- [ ] State management documented
- [ ] Common patterns documented
- [ ] Decision trees created
- [ ] Examples added
- [ ] Inline docs added

### Code Quality
- [ ] No linting errors
- [ ] No TypeScript errors
- [ ] Code review complete
- [ ] Refactoring complete

---

## Maintenance Cost Reduction Estimate

Completing Phase 3 reduces maintenance cost by approximately **50%**:

- Store unification: **Simpler mental model**
- Clear boundaries: **Fewer bugs**
- Less duplication: **Easier changes**
- Better tests: **Faster debugging**
- Good docs: **Faster onboarding**

---

## Next Steps

1. Review all tasks with engineering team
2. Estimate effort for each task
3. Assign owners
4. Begin Task 3.1 (highest impact)
5. Complete all tasks
6. Verify exit criteria
7. **Optionally** proceed to Phase 4

**Phase 4 is OPTIONAL. Only proceed if business value is clear.**

