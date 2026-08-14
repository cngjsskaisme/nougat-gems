# Mobile Native Floating Control Avoidance

## Problem

Some mobile browsers display native floating controls near the bottom of the viewport while the user scrolls. The page cannot directly hide these browser-owned controls, and they may overlap a site's Fixed Bottom Bar or FAB.

## Intent

Do not try to remove the native control. Temporarily move fixed web UI upward while overlap is likely, then return it to its normal position once activity ends.

## State

```text
browserMatchesTarget
scrollActive
uiElevated
suppressedUntil
```

Multiple fixed UI elements should share the same `uiElevated` state.

## Flow

```text
scroll input
   ↓
Suppressed?
 ├─ yes → ignore
 └─ no
      ↓
Near top?
 ├─ yes → uiElevated = false
 └─ no  → scrollActive = true
             ↓
          uiElevated = true
             ↓ idle delay
          uiElevated = false
```

## Suppression

Use short suppression windows for situations where scroll events should not be interpreted as user navigation, such as keyboard show/hide, interaction with the fixed UI itself, or Coachmark/Overlay transitions.

## Browser Detection

Enable browser-specific workarounds only in the Client Runtime. User-Agent detection is imperfect, so prefer Feature Detection when possible. When UA detection is necessary, keep its blast radius contained inside this Gem.

## Configurable Values

Treat these as configuration rather than fixed Gem constants because they may vary by product layout and browser version:

- scroll idle delay
- top reset threshold
- base bottom offset
- elevated bottom offset
- keyboard suppression duration

## Cleanup

Reset timers and scroll state on Route transitions and relevant Component lifecycle boundaries. Give event listeners a single owner so they are not registered repeatedly.

## Accessibility

Respect `prefers-reduced-motion` when moving UI, and verify that avoiding the native control does not push important Actions outside the viewport.

## Validation

- overlap is reduced while scrolling in the target browser
- behavior is a no-op in non-target browsers
- UI immediately returns to its base position near the top of the document
- keyboard/Overlay events do not create false positives
- no stale state survives Route transitions
