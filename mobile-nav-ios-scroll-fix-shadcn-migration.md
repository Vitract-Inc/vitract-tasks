# PRD — Mobile Navigation Drawer (iOS Scroll Fix + shadcn Migration)

## Overview

### Problem Statement

The current mobile navigation drawer **does not scroll correctly on iOS Safari**.
This issue has been **reproduced locally using Safari Responsive Design Mode** and affects usability on iPhone devices.

The root cause is **not Payload CMS**, but a combination of:

* Incorrect mobile drawer positioning (`absolute`)
* Missing bounded scroll container
* iOS Safari scroll quirks with sticky headers and overlays

### Goal

1. **Replace the custom mobile drawer implementation with shadcn (Radix-based) components** to standardize behavior and reduce future platform-specific issues.
2. **Explicitly fix iOS Safari scrolling behavior**, even if shadcn alone does not fully resolve it.
3. Ensure the mobile nav is **stable, scrollable, accessible, and future-proof**.

---

## Non-Goals

* Desktop navigation behavior changes (desktop dropdowns should remain functionally identical)
* Payload CMS schema changes
* Visual redesign beyond what is required for correctness

---

## Success Criteria

* Mobile navigation drawer **scrolls smoothly on iOS Safari**
* Drawer works correctly on:

  * iOS Safari
  * Android Chrome
  * Desktop browsers
* Drawer implementation follows **shadcn / Radix best practices**
* No body scroll when drawer is open
* No regression in navigation functionality

---

## Technical Strategy (High Level)

### Phase 1 — Migrate Drawer to shadcn

Use **shadcn’s `Sheet` component** (Radix Dialog under the hood) for:

* Correct portal behavior
* Proper overlay stacking
* Safer scroll locking
* iOS-tested interaction patterns

### Phase 2 — Explicit iOS Scroll Hardening

Even after migration, explicitly apply iOS-safe scroll rules:

* Dedicated scroll container
* Fixed positioning
* Bounded height
* Momentum scrolling

### Phase 3 — Validation & Fallback Fixes

If scrolling still fails, apply additional targeted fixes (listed below).

---

## Detailed Requirements

### 1. Replace Custom Mobile Drawer with shadcn `Sheet`

#### Requirements

* Install shadcn if not already present
* Use `Sheet`, `SheetTrigger`, `SheetContent`
* Drawer must open from the top or side (top is acceptable for nav)
* Drawer must render in a **portal** (Radix default)
* Existing nav items (`leftNav`, `rightNav`, `cta`) must remain unchanged logically

#### Notes for Developer

* **Do not reimplement custom overlay or positioning logic**
* Use shadcn defaults first, customize only when necessary

---

### 2. Scroll Container Must Be Explicit

Inside `SheetContent`:

* There must be **one explicit scroll container**
* Scroll must **not rely on `body`**
* Scroll container must:

  * Have bounded height
  * Use momentum scrolling

#### Required styles:

```css
overflow-y: auto;
-webkit-overflow-scrolling: touch;
overscroll-behavior: contain;
```

Use:

* `h-[100dvh]` or `max-h-[100dvh]`
* Avoid `100vh`

---

### 3. Drawer Positioning Rules

* Drawer must be `position: fixed`
* No `absolute` positioning for the mobile drawer
* Drawer must not be nested inside sticky or transformed parents

---

### 4. Body Scroll Lock

When drawer is open:

* Background page must not scroll
* Only the drawer content should scroll

Use:

* Radix default scroll lock OR
* Explicit `body { overflow: hidden }` as fallback

---

### 5. Sticky Header & Banner Compatibility

The project has:

* Sticky `PreOrderBanner`
* Sticky `Header`

Ensure:

* Drawer overlays **above** sticky elements
* Sticky elements do **not interfere** with drawer scrolling
* `z-index` layering is intentional and documented

---

## Validation & Testing Requirements

### Required Test Environments

* Safari → Responsive Design Mode → iPhone
* (If available) Physical iPhone Safari
* Android Chrome
* Desktop Chrome/Safari

### Required Test Scenarios

* Open drawer
* Scroll full nav list
* Open dropdown items inside drawer
* Tap nav links
* Close drawer
* Rotate device (optional but recommended)

---

## Fallback Fixes (If Issue Persists After shadcn)

If iOS scrolling is **still broken**, apply the following **in order**, testing after each step:

1. **Ensure scroll container is NOT inside an element with**

   * `transform`
   * `filter`
   * `backdrop-filter`

2. **Move scroll container one level higher**

   * Sometimes Radix content wrappers need adjustment

3. **Force scroll isolation**

   ```css
   touch-action: pan-y;
   ```

4. **Remove any `onTouchMove` or `preventDefault` logic**

   * Especially on document/body

5. **Log and inspect**

   * Confirm which element is actually scrollable in Safari DevTools

If after all of the above the issue persists:

* Escalate with screenshots + Safari inspector findings

---

## Deliverables

* Updated mobile nav implementation using shadcn
* Removal of old custom mobile drawer code
* Verified iOS scroll behavior
* Short developer note explaining:

  * Why shadcn was chosen
  * What iOS-specific fixes were applied

---

## Checklist (For PM Task Creation)

### Migration

* [ ] Install and configure shadcn (if not present)
* [ ] Replace custom mobile drawer with `Sheet`
* [ ] Ensure nav logic is preserved

### Scroll Fix

* [ ] Drawer uses `position: fixed`
* [ ] Dedicated scroll container exists
* [ ] `overflow-y: auto` applied
* [ ] `-webkit-overflow-scrolling: touch` applied
* [ ] `100dvh` used instead of `100vh`
* [ ] Body scroll locked when drawer is open

### Compatibility

* [ ] Works with sticky header and banner
* [ ] Correct z-index layering
* [ ] No background scroll bleed

### Testing

* [ ] Tested in Safari Responsive Design Mode
* [ ] Tested on iOS Safari (if available)
* [ ] Tested on Android Chrome
* [ ] Desktop unaffected

### Fallbacks

* [ ] Verified no parent `transform` issues
* [ ] Scroll container isolation confirmed
* [ ] Escalation notes added if unresolved

---

## Final Note to Developer

This task is **not just a refactor**.
It is a **platform compatibility fix** with explicit iOS requirements.
Do not assume that visual correctness implies scroll correctness — **Safari must be explicitly verified**.
