# Frontend Health Info Journey - Phased Implementation Plan

**Date:** 2026-02-10  
**Scope:** End-to-end health information journey refactoring  
**Philosophy:** Risk-first, safety-driven implementation

---

## Overview

This plan breaks down the frontend refactoring into **4 distinct phases**, ordered by **risk severity** and **user impact**.

The phasing is designed to:
1. **Eliminate data corruption and user-facing bugs FIRST**
2. **Standardize UX flows SECOND**
3. **Improve maintainability THIRD**
4. **Add optional enhancements LAST**

---

## Risk-First Philosophy

### Why Phase 1 Blocks Everything Else

**Phase 1 is non-negotiable** because it addresses issues that can:
- Cause users to submit incorrect health information
- Lose or corrupt user data
- Break core user flows (refresh, navigation, back button)
- Leak data between kits
- Silently fail without user awareness

**No other work should proceed until Phase 1 is complete.**

Rationale:
- Building UX improvements on top of broken state management wastes effort
- Refactoring stores while data leakage exists compounds the problem
- Adding features to flows that can corrupt data increases risk surface

---

## Phase Definitions

### Phase 1: Risk Elimination (MANDATORY)
**Goal:** Eliminate all data corruption, state leakage, and broken user flows

**Scope:**
- Fix cross-kit data leakage in Zustand stores
- Fix broken refresh/navigation flows (location.state dependency)
- Fix dual-store inconsistencies
- Fix missing edit enforcement (submitted status)
- Fix unsafe cache behavior

**Exit Criteria:**
- No kit can see another kit's data
- All pages work correctly after refresh
- Edit flows respect submitted status
- Store state is predictable and isolated

**Estimated Risk Reduction:** 90% of critical bugs eliminated

---

### Phase 2: UX Standardization (REQUIRED)
**Goal:** Align user flows with industry-standard patterns

**Scope:**
- Remove Step 3 (replace with toast/modal)
- Standardize edit/view separation
- Fix navigation patterns
- Improve error states and loading UX
- Ensure consistent flow across practitioner/client paths

**Exit Criteria:**
- No unnecessary full-page confirmations
- Clear edit vs view distinction
- Predictable navigation behavior
- Consistent UX patterns across roles

**Estimated UX Improvement:** 70% reduction in user confusion

---

### Phase 3: Maintainability (REQUIRED)
**Goal:** Reduce technical debt and improve code quality

**Scope:**
- Unify dual Zustand stores
- Establish clear state boundaries
- Reduce component duplication
- Improve testability
- Document state management patterns

**Exit Criteria:**
- Single source of truth for health info state
- Clear component responsibilities
- Testable state transitions
- Documented architecture

**Estimated Maintenance Cost Reduction:** 50% fewer bugs in future changes

---

### Phase 4: Optional Enhancements (OPTIONAL)
**Goal:** Add collaborative editing polish

**Scope:**
- Multi-user presence indicators
- Real-time conflict warnings
- Optimistic UI updates
- Enhanced SSE collaboration UX

**Exit Criteria:**
- (Optional - only if business value is clear)

**Note:** This phase should only be pursued if:
- Phases 1-3 are complete
- Business case is validated
- Resources are available

---

## Execution Rules

### Sequential Execution (STRICT)
1. Complete Phase 1 → Test → Deploy
2. Complete Phase 2 → Test → Deploy
3. Complete Phase 3 → Test → Deploy
4. (Optional) Phase 4

**Do NOT:**
- Mix tasks from different phases
- Skip Phase 1 tasks
- Start Phase 2 before Phase 1 is complete
- Treat Phase 1 as "nice to have"

---

### Decision Framework: "Which Phase Does This Belong In?"

**Ask these questions in order:**

1. **Can this cause data corruption, loss, or leakage?**
   - YES → Phase 1
   - NO → Continue

2. **Can this break a core user flow (submit, edit, view)?**
   - YES → Phase 1
   - NO → Continue

3. **Does this rely on unsafe assumptions (e.g., location.state)?**
   - YES → Phase 1
   - NO → Continue

4. **Is this a UX pattern that confuses users or violates standards?**
   - YES → Phase 2
   - NO → Continue

5. **Is this about code organization, duplication, or testability?**
   - YES → Phase 3
   - NO → Continue

6. **Is this a collaborative editing enhancement?**
   - YES → Phase 4 (Optional)
   - NO → Re-evaluate

**When in doubt → Phase 1**

---

## Safety & Correctness Definitions

### Frontend Safety
- **No cross-kit data leakage:** Kit A's answers never appear in Kit B
- **Refresh-safe:** All pages work correctly after browser refresh
- **Navigation-safe:** Back button, direct links, and route changes work predictably
- **Store isolation:** Each kit's state is independently managed
- **Error boundaries:** Failed API calls don't crash the UI

### Frontend Correctness
- **Single source of truth:** One authoritative data source per entity
- **Predictable state transitions:** State changes follow documented patterns
- **Validation enforcement:** Business rules (e.g., submitted status) are enforced
- **Consistent UX:** Same patterns across practitioner/client flows
- **Accessible state:** No hidden or unreachable states

---

## File Structure

```
assessments/health_info_frontend_phased_plan_2026-02-10/frontend/
├── README.md (this file)
├── PHASE_1_RISK_ELIMINATION.md
├── PHASE_2_UX_STANDARDIZATION.md
├── PHASE_3_MAINTAINABILITY.md
└── PHASE_4_OPTIONAL_ENHANCEMENTS.md
```

---

## Next Steps

1. Read `PHASE_1_RISK_ELIMINATION.md` in full
2. Review each task's acceptance criteria
3. Estimate effort for Phase 1 only
4. Begin implementation (Phase 1, Task 1)
5. Do NOT proceed to Phase 2 until Phase 1 exit criteria are met

---

## Questions?

If you're unsure whether a task belongs in Phase 1:
- **Default to Phase 1**
- Ask: "Can this corrupt user data or break a flow?"
- If yes → Phase 1
- If no → Re-read the decision framework

**Remember:** Phase 1 is about user safety. When in doubt, prioritize safety.

