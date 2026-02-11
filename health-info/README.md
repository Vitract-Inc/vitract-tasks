# Health Info - Phased Implementation Plans

**Date:** 2026-02-10  
**Scope:** Coordinated backend and frontend remediation for the Health Information journey  
**Philosophy:** Risk-first, safety-driven execution

## Purpose

This directory is the **central index** for the Health Information remediation plans. It provides a **shared, phase-based roadmap** for both backend and frontend work, ordered by **risk severity**:

1. **Phase 1: Risk Elimination** (mandatory, block other work)
2. **Phase 2: Standardization / UX Consistency**
3. **Phase 3: Maintainability / Scalability**
4. **Phase 4: Optional Enhancements**

The goal is to eliminate data loss and correctness issues **before** making UX or refactoring improvements.

## How To Use This Plan

Start with the **Phase 1 documents** for both backend and frontend. Phase 1 items are non-negotiable and must be completed before any Phase 2+ work begins.

Recommended order:
1. Read `health-info/backend/README.md` (backend risks and constraints)
2. Read `health-info/frontend/README.md` (frontend risks and constraints)
3. Execute Phase 1 items in each area, then continue in sequence

## Getting Started (Day 0)

Use this checklist to align quickly before implementation begins:
1. Confirm the **current risk status** with stakeholders (data loss, compliance, user impact).
2. Agree on **Phase 1 scope** and sequencing across backend and frontend.
3. Assign **owners** for each Phase 1 task (backend + frontend).
4. Review **deployment and rollback** expectations for Phase 1 changes.
5. Set up a **status cadence** (daily updates until Phase 1 completion).

## Repository Structure

```
health-info/
├── README.md
├── backend/
└── frontend/
```

## Backend Plan Index

Key documents in `health-info/backend/`:
- `health-info/backend/README.md` - Backend phased plan, risks, constraints
- `health-info/backend/QUICK_START.md` - Day-by-day kickoff guidance
- `health-info/backend/IMPLEMENTATION_SUMMARY.md` - Executive summary
- `health-info/backend/PROGRESS_CHECKLIST.md` - Phase completion tracking
- `health-info/backend/PHASE_1_RISK_ELIMINATION.md`
- `health-info/backend/PHASE_2_STANDARDIZATION.md`
- `health-info/backend/PHASE_3_MAINTAINABILITY.md`
- `health-info/backend/PHASE_4_OPTIONAL_ENHANCEMENTS.md`

## Frontend Plan Index

Key documents in `health-info/frontend/`:
- `health-info/frontend/README.md` - Frontend phased plan, risk framing
- `health-info/frontend/PHASE_1_RISK_ELIMINATION.md`
- `health-info/frontend/PHASE_2_UX_STANDARDIZATION.md`
- `health-info/frontend/PHASE_3_MAINTAINABILITY.md`
- `health-info/frontend/PHASE_4_OPTIONAL_ENHANCEMENTS.md`

## Execution Rules (Non-Negotiable)

- **Phase 1 blocks all other phases** in both backend and frontend
- **No cross-phase mixing** (finish a phase, test, deploy, then proceed)
- **Database is source of truth** for backend changes
- **Frontend state isolation and refresh safety** must be guaranteed before UX work

## Status Tracking

Use the phase-specific checklists and review docs to record completion:
- Backend: `health-info/backend/PROGRESS_CHECKLIST.md` and `health-info/backend/PHASE_1_RISK_ELIMINATION_REVIEW.md`
- Frontend: `health-info/frontend/PHASE_1_RISK_ELIMINATION_REVIEW.md`

## Questions / Updates

If a change affects scope or sequencing:
1. Update the relevant phase document
2. Update `health-info/backend/IMPLEMENTATION_SUMMARY.md` or `health-info/frontend/README.md`
3. Note the change date at the top of the modified file
