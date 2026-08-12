# Samsung Internet Native "Scroll to Top" Button Overlap Avoidance Logic
## Program Design Document (Reusable Unit)

---

## 1. Document Overview

| Item | Content |
|------|------|
| Document name | Samsung Internet native floating button overlap avoidance logic design document |
| Version | 1.0 |
| Scope | Mobile web projects with fixed bottom UI using Vue 3 / Nuxt |
| Reusable unit | 3 composables + 1 application guide (project-independent) |
| Prerequisites | Vue 3 Composition API, Nuxt `useState`/`useRouter`, SSR environment support |

---

## 2. Problem Definition

### 2.1 Background
- Samsung Internet browser displays a native "Scroll to top" floating button at the **bottom center** of the screen when scrolling through a document.
- This button cannot be hidden or disabled via web APIs (`CSS`/`JS`).
- Visual and operational overlap occurs with the service's fixed bottom UI (chat bar, FAB, follow-up question cards, etc.).

### 2.2 Solution Strategy
- Since the native button cannot be removed, **raise the web fixed UI upward only during scroll activity** to avoid overlap.
- After scrolling stops and a specified delay period elapses, return to the original position.
- Near the top of the document (where the native button is not displayed), return to the original position immediately without delay.

### 2.3 Requirements

| ID | Requirement |
|----|----------|
| REQ-01 | Detect Samsung Internet browser environment via UA-based detection |
| REQ-02 | Raise bottom UI with a top offset during scroll activity |
| REQ-03 | Return to original position after the specified delay period elapses following scroll stop |
| REQ-04 | Return to original position immediately without delay when within the document top threshold |
| REQ-05 | All logic should operate as no-op in non-Samsung environments |
| REQ-06 | Reset the raised state and delay timer on route transition |
| REQ-07 | Multiple fixed bottom UI components should share the same state (singleton) |
| REQ-08 | Scroll caused by chat bar/keyboard interaction should not be considered as activity |
| REQ-09 | Be able to suppress scroll activity detection while overlays such as coachmarks are displayed |
| REQ-10 | Operate safely in SSR environments (detect after client transition) |

---

## 3. Module Composition

### 3.1 Module Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│  Application component layer (project-specific implementation)│
│  - Fixed bottom UI components bind computed classes          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  Module 1: useSamsungChatChrome (orchestration)              │
│  - State management, watch/router hook registration,         │
│    computed output                                           │
└───────┬──────────────────────────────┬──────────────────────┘
        │                              │
┌───────▼──────────────────────┐  ┌───▼───────────────────────┐
│ Module 2: useScrollChrome    │  │ Module 3: useSamsungInternet│
│ Visibility                   │  │ Browser                     │
│ - Provides scrollActive state│  │ - UA detection              │
│ - Integrated scroll event    │  │ - isSamsungInternet ref     │
│   handling                   │  │                             │
└──────────────────────────────┘  └────────────────────────────┘
```

### 3.2 Module List

| Module | Recommended filename | Role | Dependencies |
|------|---------------|------|------|
| Module 1 | `useSamsungChatChrome.ts` | Raised state management, watch/router hooks, computed output | Module 2, 3 |
| Module 2 | `useScrollChromeVisibility.ts` | Scroll activity detection, `scrollActive` state provision, suppression API | None |
| Module 3 | `useSamsungInternetBrowser.ts` | UA-based Samsung Internet detection | None |

---

## 4. Module Detailed Design

### 4.1 Module 3 — `useSamsungInternetBrowser`

#### 4.1.1 Interface

```ts
function useSamsungInternetBrowser(): {
  isSamsungInternet: Ref<boolean>
}
```

#### 4.1.2 Design Specification

| Item | Content |
|------|------|
| Purpose | Detect Samsung Internet browser environment |
| Detection method | `navigator.userAgent` regex matching |
| Detection regex | `/SamsungBrowser/i` |
| Detection timing | `onMounted` (client-side) |
| Initial value | `false` |
| SSR behavior | Maintain initial value when `typeof navigator === 'undefined'` |

#### 4.1.3 Processing Order

1. Initialize `isSamsungInternet` ref to `false`
2. Execute `onMounted` hook
3. If `navigator` does not exist (SSR), exit immediately
4. Evaluate whether `navigator.userAgent` contains `SamsungBrowser`
5. On match, set `isSamsungInternet.value = true`

#### 4.1.4 Reference Specifications
- Samsung Internet UA format: `Mozilla/5.0 ... SamsungBrowser/<version> ...`
- Regex is case-insensitive (`/i` flag) to handle variations

---

### 4.2 Module 2 — `useScrollChromeVisibility`

#### 4.2.1 Interface

```ts
function useScrollChromeVisibility(): {
  chromeVisible: Ref<boolean>           // Backward compatibility (always true)
  scrollActive: Ref<boolean>            // Whether scroll is active
  suppressScrollChromeForChatBarMs: (ms?: number) => void
  suppressScrollChromeForKeyboardMs: (ms?: number) => void
}

// Module external export functions
function isWindowScrollWithinTopResetPx(): boolean
function setCoachmarkScrollChromeBlocked(blocked: boolean): void
```

#### 4.2.2 Constant Definitions

| Constant name | Value | Description |
|--------|-----|------|
| `SCROLL_IDLE_MS` | 150ms | Scroll stop determination time (if no additional scroll events occur during this time, considered inactive) |
| `VIEWPORT_SCROLL_IGNORE_MULTIPLIER` | 0.75 | Top ignore area multiplier. In the area where `scrollY < innerHeight * 0.75`, scroll is not considered as activity |
| `SCROLL_TOP_ACTIVE_RESET_PX` | 200px | Top forced deactivation threshold. If `scrollY <= 200`, `scrollActive` is always forced to `false` |
| `VV_RESIZE_SUPPRESS_DEBOUNCE_MS` | 220ms | `visualViewport.resize` debounce time |
| `VV_RESIZE_SUPPRESS_DURATION_MS` | 850ms | Scroll suppression duration caused by keyboard |

#### 4.2.3 State

| State name | Type | Scope | Description |
|--------|------|--------|------|
| `chromeVisible` | `boolean` | `useState` | For backward compatibility. Always `true` |
| `scrollActive` | `boolean` | `useState` | Whether scroll is active. Watched by Module 1 |
| `suppressScrollChromeUntil` | `number` | Module variable | Suppression end timestamp |
| `coachmarkBlocksScrollChrome` | `boolean` | Module variable | Suppression flag during overlay display |
| `idleTimer` | `setTimeout return \| null` | Module variable | Scroll stop determination timer |
| `vvResizeSuppressDebounce` | `setTimeout return \| null` | Module variable | vv.resize debounce timer |
| `listenerAttached` | `boolean` | Module variable | Prevent duplicate listener registration |
| `routeHooksRegistered` | `boolean` | Module variable | Prevent duplicate router hook registration |

#### 4.2.4 Core Processing — `markScrollActivity()`

| Step | Condition | Action |
|------|------|------|
| 1 | `coachmarkBlocksScrollChrome == true` | Exit (suppressed) |
| 2 | `scrollY <= SCROLL_TOP_ACTIVE_RESET_PX` | Cancel `idleTimer`, `scrollActive = false`, exit |
| 3 | `Date.now() < suppressScrollChromeUntil` | Exit (suppressed) |
| 4 | `scrollY < innerHeight * VIEWPORT_SCROLL_IGNORE_MULTIPLIER` | Exit (top area ignored) |
| 5 | (All above conditions not met) | `scrollActive = true`, reset `idleTimer` (set to `false` after `SCROLL_IDLE_MS`), `chromeVisible = true` |

#### 4.2.5 Event Listeners (registered once, `listenerAttached` flag)

| Target | Event | Options | Handler | Purpose |
|------|--------|------|--------|------|
| `window` | `scroll` | passive | `markScrollActivity` | Document-wide scroll |
| `document` | `scroll` | capture, passive | `markScrollActivity` | Internal overflow scroll (child elements) |
| `window` | `wheel` | passive | `markScrollActivity` | Mouse wheel/trackpad |
| `window` | `touchmove` | passive | `markScrollActivity` | Touch scroll/inertia |
| `visualViewport` | `scroll` | passive | `markScrollActivity` | Layout viewport scroll (address bar, etc.) |
| `visualViewport` | `resize` | passive | `onVisualViewportResizeForKeyboard` | Soft keyboard show/hide |
| `document` | `focusin` | capture | `onFocusInMaybeKeyboard` | Suppress on input element focus |
| `document` | `focusout` | capture | `onFocusOutMaybeKeyboard` | Suppress on input element blur |

#### 4.2.6 Router Hooks (registered once, `routeHooksRegistered` flag)

| Hook | Action |
|----|------|
| `router.afterEach` | `nextTick(() => resetScrollActivity())` |

**`resetScrollActivity()` processing**
1. Cancel `idleTimer`
2. Cancel `vvResizeSuppressDebounce`
3. `scrollActive = false`
4. `chromeVisible = true`

#### 4.2.7 External API Functions

| Function | Signature | Purpose |
|------|----------|------|
| `suppressScrollChromeForChatBarMs` | `(ms = 600) => void` | Ignore scroll after fixed UI interaction |
| `suppressScrollChromeForKeyboardMs` | `(ms = 1000) => void` | Ignore scroll after keyboard show/hide |
| `setCoachmarkScrollChromeBlocked` | `(blocked: boolean) => void` | Suppress scroll activity detection during overlay display |
| `isWindowScrollWithinTopResetPx` | `() => boolean` | Check `scrollY <= 200`. Shared threshold with Module 1 |

#### 4.2.8 Suppression Mechanism Details

**`suppressScrollChromeUntil`-based suppression**
- `bumpSuppressChromeUntil(ms)`: `suppressScrollChromeUntil = max(current value, Date.now() + ms)`
- In `markScrollActivity()`, if `Date.now() < suppressScrollChromeUntil`, activity processing is skipped
- Suppression is time-based, so consecutive calls do not push the end time back; it is corrected with `max`

**`visualViewport.resize` debounce**
- `resize` events fire consecutively, so suppression is applied once after a 220ms debounce
- Suppression duration: 850ms

**Focus-based keyboard suppression**
- If the target of `focusin`/`focusout` is `input, textarea, select` or `contenteditable`, suppress for 850ms

---

### 4.3 Module 1 — `useSamsungChatChrome`

#### 4.3.1 Interface

```ts
function useSamsungChatChrome(): {
  floatingBarBottomClass: ComputedRef<string>
  suggestedCardBottomClass: ComputedRef<string>
  samsungBarElevated: Ref<boolean>
}
```

#### 4.3.2 Constant Definitions

| Constant name | Value | Description |
|--------|-----|------|
| `SAMSUNG_LOWER_DELAY_MS` | 1800ms | Delay time for lowering (return to original position) after scroll stop |

#### 4.3.3 State

| State name | Type | Scope | Description |
|--------|------|--------|------|
| `samsungBarElevated` | `boolean` | `useState('samsung-chat-chrome-elevated')` | Whether bottom UI is raised. Shared across multiple components |
| `samsungLowerTimer` | `setTimeout return \| null` | Module variable | Lowering delay timer |
| `samsungChromeWatchRegistered` | `boolean` | Module variable | Prevent duplicate watch/listener registration |

#### 4.3.4 Initialization Processing (once, `samsungChromeWatchRegistered` flag)

**Prerequisite**: Only executes when `import.meta.client && !samsungChromeWatchRegistered`

**A. Scroll listener registration (for immediate lowering near top)**

| Target | Event | Options | Handler |
|------|--------|------|--------|
| `window` | `scroll` | passive | `lowerSamsungBarIfNearTopFromScroll` |
| `visualViewport` | `scroll` | passive | `lowerSamsungBarIfNearTopFromScroll` |

**`lowerSamsungBarIfNearTopFromScroll()` processing**

| Step | Condition | Action |
|------|------|------|
| 1 | `isSamsungInternet == false` | Exit |
| 2 | `isWindowScrollWithinTopResetPx() == false` | Exit |
| 3 | (Conditions met) | `clearSamsungLowerTimer()`, `samsungBarElevated = false` |

**B. watch: `scrollActive`**

| `scrollActive` value | Additional condition | Action |
|-------------------|-----------|------|
| `true` | - | `clearSamsungLowerTimer()`, `samsungBarElevated = true` (raise immediately) |
| `false` | `isWindowScrollWithinTopResetPx() == true` | `clearSamsungLowerTimer()`, `samsungBarElevated = false` (lower immediately) |
| `false` | `isWindowScrollWithinTopResetPx() == false` | `samsungLowerTimer = setTimeout(lower, SAMSUNG_LOWER_DELAY_MS)` (delayed lowering) |

**C. watch: `isSamsungInternet`**

| Value | Action |
|----|------|
| `true` | Exit (no-op) |
| `false` | `clearSamsungLowerTimer()`, `samsungBarElevated = false` (cleanup on non-Samsung transition) |

**D. Router hook**

| Hook | Action |
|----|------|
| `router.afterEach` | `nextTick(() => { clearSamsungLowerTimer(); samsungBarElevated = false })` |

#### 4.3.5 Helper Functions

| Function | Processing |
|------|-----------|
| `clearSamsungLowerTimer()` | If `samsungLowerTimer` exists, `clearTimeout` then assign `null` |

#### 4.3.6 Output computed

**`floatingBarBottomClass`** — for fixed bottom bar

| Condition | Return value | Meaning |
|------|--------|------|
| `isSamsungInternet == false` | `'bottom-7'` | Default position |
| `isSamsungInternet == true && samsungBarElevated == true` | `'bottom-16'` | Raised (avoids native button) |
| `isSamsungInternet == true && samsungBarElevated == false` | `'bottom-7'` | Returned |

**`suggestedCardBottomClass`** — for follow-up UI card

| Condition | Return value | Meaning |
|------|--------|------|
| `isSamsungInternet == true && samsungBarElevated == true` | `'bottom-[calc(6rem+2.25rem)]'` | Raised (adds same delta of 2.25rem as bar) |
| Otherwise | `'bottom-[6rem]'` | Default position |

> **Note**: `2.25rem` matches the difference between `bottom-16` and `bottom-7` (`4rem - 1.75rem = 2.25rem`). Ensures the fixed bar and follow-up card maintain the same spacing.

---

## 5. State Transition Specification

### 5.1 `samsungBarElevated` State Transitions

| Current state | Event | Condition | Next state | Delay |
|-----------|--------|------|-----------|------|
| `false` | `scrollActive: false → true` | Samsung | `true` | None (immediate) |
| `true` | `scrollActive: true → false` | Samsung, `scrollY <= 200` | `false` | None (immediate) |
| `true` | `scrollActive: true → false` | Samsung, `scrollY > 200` | `false` | After 1800ms |
| `true` | `scroll` event | Samsung, `scrollY <= 200` | `false` | None (immediate) |
| `true` | `isSamsungInternet: true → false` | - | `false` | None (immediate) |
| `true` | `router.afterEach` | - | `false` | Immediate after `nextTick` |
| `false` | (All above events) | - | `false` | - (No change) |

### 5.2 `scrollActive` State Transitions

| Current state | Event | Condition | Next state | Timer |
|-----------|--------|------|-----------|--------|
| `false` | Scroll input | Not suppressed, not in top ignore area, `scrollY > 200` | `true` | Set `idleTimer` 150ms |
| `true` | `idleTimer` expires | - | `false` | - |
| `true`/`false` | Scroll input | `scrollY <= 200` | `false` | Cancel `idleTimer` |
| `true`/`false` | Scroll input | During suppression | (No change) | - |
| `true`/`false` | Scroll input | `scrollY < innerHeight * 0.75` | (No change) | - |
| `true`/`false` | Scroll input | `coachmarkBlocksScrollChrome == true` | (No change) | - |
| `true`/`false` | `router.afterEach` | - | `false` | Cancel `idleTimer` |

---

## 6. Application Guide

### 6.1 Application Component Requirements

Fixed bottom UI components must satisfy the following.

| Requirement | Description |
|------|------|
| `position: fixed` | Fixed bottom position |
| `bottom` offset class binding | Bind Module 1's computed to `:class` |
| `transition-[bottom]` | Smooth transition on offset change (recommended) |
| `motion-reduce:transition-none` | `prefers-reduced-motion` support (recommended) |

### 6.2 Application Example

```vue
<script setup lang="ts">
import { useSamsungChatChrome } from '~/composables/useSamsungChatChrome'

const { floatingBarBottomClass } = useSamsungChatChrome()
</script>

<template>
  <div
    class="fixed left-0 right-0 transition-[bottom] duration-300 ease-out motion-reduce:transition-none"
    :class="floatingBarBottomClass"
  >
    <!-- Fixed bottom UI content -->
  </div>
</template>
```

### 6.3 Notes on Multiple Component Application

- `samsungBarElevated` is `useState`-based, so all components using the **same key** share the state.
- watch/listeners are **registered only once** via the module variable flag (`samsungChromeWatchRegistered`).
- Even if multiple components call `useSamsungChatChrome()`, there are no side effects.

### 6.4 Offset Value Customization

| Change target | Location | Description |
|-----------|------|------|
| Default/raised offset | Module 1 computed return value | Adjust `bottom-7` / `bottom-16` to fit project layout |
| Follow-up UI offset | Module 1 computed return value | Adjust `bottom-[6rem]` / `bottom-[calc(6rem+2.25rem)]` |
| Raise delta | `2.25rem` | Ensure `bottom-16 - bottom-7` difference matches follow-up card delta |
| Lowering delay | `SAMSUNG_LOWER_DELAY_MS` | Adjust to match native button dismissal timing |
| Top threshold | `SCROLL_TOP_ACTIVE_RESET_PX` | Exported from Module 2. Must share the same value with Module 1 |

### 6.5 Suppression API Usage Guide

| Scenario | Function to call | Timing |
|----------|----------|------|
| Immediately after fixed UI button click | `suppressScrollChromeForChatBarMs(600)` | `pointerdown` / `click` handler |
| Input element focus/blur | `suppressScrollChromeForKeyboardMs(850)` | Auto-applied (focus listener) |
| Overlay/coachmark display | `setCoachmarkScrollChromeBlocked(true)` | On overlay open |
| Overlay/coachmark dismissal | `setCoachmarkScrollChromeBlocked(false)` | On overlay close |

---

## 7. Constraints and Notes

| ID | Content |
|----|------|
| CON-01 | Since detection is UA-based, misdetection is possible if Samsung Internet changes its UA string or another browser includes `SamsungBrowser` |
| CON-02 | In environments that do not support the `visualViewport` API (older browsers), vv-based scroll detection and keyboard suppression will not work (null check safety guard) |
| CON-03 | Module variables (`listenerAttached`, etc.) are initialized once per module load unit. In HMR environments, duplicate registration may occur |
| CON-04 | `useState` key strings must be globally unique across the project to avoid conflicts (`'samsung-chat-chrome-elevated'`, `'scroll-chrome-active'`, `'scroll-chrome-visible'`) |
| CON-05 | Raise/lower offset classes (`bottom-7`, `bottom-16`) are based on Tailwind CSS. When using a different CSS framework, adjust the computed return values |
| CON-06 | `SCROLL_TOP_ACTIVE_RESET_PX` (200px) is a threshold shared between Module 1 and Module 2. Export from a single source to ensure consistency |
| CON-07 | The exact scroll threshold for the native "Scroll to top" button's display/dismissal may vary by Samsung Internet version. 200px is an empirical approximation |

---

## 8. Verification Items

| ID | Item | Method |
|----|------|------|
| VER-01 | Does the fixed UI raise when scrolling in Samsung Internet? | Change UA on real device/emulator then scroll |
| VER-02 | Does it return to original position 1800ms after scroll stop? | Measure timing after scrolling |
| VER-03 | Does it return to original position immediately within top 200px? | Check after `scrollTo(0, 100)` |
| VER-04 | Is it no-op in non-Samsung environments? | `samsungBarElevated` always `false` in Chrome/Safari |
| VER-05 | Is state reset on route transition? | `samsungBarElevated == false` after page navigation |
| VER-06 | Do multiple components share the same state? | Offset sync when two components render simultaneously |
| VER-07 | Does it not misidentify keyboard display as scroll activity? | No `scrollActive` change after input focus |
| VER-08 | Does it render without errors in SSR environment? | Check page source after SSR build |
| VER-09 | Is scroll activity suppressed during coachmark display? | `scrollActive == false` when scrolling with overlay open |
| VER-10 | Is transition omitted with `prefers-reduced-motion`? | Confirm `transition` not applied after OS setting |

---

## 9. File Structure (Reusable Package)

```
composables/
├── useSamsungInternetBrowser.ts   # Module 3: UA detection (no dependencies)
├── useScrollChromeVisibility.ts    # Module 2: Scroll activity detection (no dependencies)
└── useSamsungChatChrome.ts         # Module 1: Orchestration (depends on Module 2, 3)
```

**Dependency graph**

```
useSamsungChatChrome
  ├── useScrollChromeVisibility
  │     └── (isWindowScrollWithinTopResetPx export)
  └── useSamsungInternetBrowser
```

**External dependencies**: Vue 3 (`ref`, `computed`, `watch`, `nextTick`, `onMounted`), Nuxt (`useState`, `useRouter`, `import.meta.client`/`import.meta.server`)

---
