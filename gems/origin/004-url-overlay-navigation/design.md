# URL-based UI Action / Overlay Navigation

## 1. Intent

Represent UI states in the URL when they are meaningful to the user and worth restoring through refresh, deep links, or Browser Back/Forward. Ephemeral state such as input drafts, loading, focus, or animation state should stay out of the URL.

## 2. Query Contract

```text
?action=<ACTION>&payload=<PAYLOAD>&meta=<META>
```

- `action`: identity of the current UI state
- `payload`: target data required by that state
- `meta`: auxiliary context such as version, entry source, or presentation

Use generalized examples rather than domain-specific names.

```text
?action=profile.view
?action=item.preview&payload=...
?action=form.edit&payload=...
?action=confirm.discard
```

Do not use Component names or function names as Action IDs.

## 3. State Layers

```text
Navigable UI State  → URL
Domain Session State → feature-local state
UI Infrastructure   → overlay infrastructure
Analytics Event      → event pipeline
```

The URL answers **where the user is now**. An Event answers **what action led there**.

## 4. Action Registry

```ts
interface ActionDefinition<TPayload = unknown> {
  action: string
  presentation: 'page' | 'overlay' | 'sheet' | 'dialog' | 'inline'
  payload?: {
    schema: Schema<TPayload>
    encrypted?: boolean
  }
  navigation: NavigationPolicy
}
```

Presentation defines how a state is displayed. Navigation defines how it behaves in Browser History.

## 5. Navigation Policy

```ts
interface NavigationPolicy {
  enter: 'push' | 'replace'
  exit: 'back' | 'restore' | 'replace' | 'custom'
  lifetime: 'persistent' | 'ephemeral'
  directEntryExit?: 'fallback' | 'replace' | 'custom'
}
```

### Persistent

After leaving via Back, Forward may restore the same UI state.

### Ephemeral

Do not try to physically delete Browser History entries. Instead, track whether the state has already been consumed in a **Logical UI History** and normalize to the parent state if the user later reaches that consumed entry through Forward.

```text
Physical Browser History ≠ Logical UI History
```

## 6. Dispatcher / Resolver Symmetry

```text
Component
  ↓ ui.dispatch(action, payload)
Registry + Validation
  ↓ encode
Router
  ↓ URL
Resolver
  ↓ decode + validate
Resolved UI State
```

Feature Components express intent to a central Navigation Manager instead of deciding `router.push()` / `router.back()` policies themselves.

## 7. Direct Entry

If an Overlay was opened from inside the app, Close may safely use `back`. If the user entered directly through an Overlay URL, `back` may leave the application entirely. Define a separate `directEntryExit` policy for that case.

## 8. Sensitive Payload

URLs may be retained in History, logs, and referrers. Secrets, tokens, passwords, and credentials must never be placed in a URL, even when encrypted.

If a sensitive identifier must be represented in the URL, minimize it and consider Authenticated Encryption plus URL-safe encoding. Server-side authorization is still required independently.

## 9. Overlay Infrastructure

A shared Overlay Shell owns backdrop behavior, ESC handling, scroll lock, focus trap/restore, transitions, z-index, and close requests. Feature code focuses on display data and domain state.

## 10. Validation

- unknown action → fallback
- payload decode/schema failure → fallback or safe error state
- distinguish direct entry from in-app navigation
- prevent Forward from reactivating a consumed ephemeral state
- avoid duplicate focus/scroll cleanup
- verify that no Secrets are placed in URLs
