# NDJSON × json-render Adapter — Frontend

## Core Boundary

Do not connect the NDJSON Parser directly to the UI Renderer.

```text
NDJSON Event
  ↓ Contract + Sequence Guard
Part Store
  ↓
UI Patch Compiler
  ↓
workingSpec
  ↓ Boundary Validation
renderSpec (Last-Known-Good)
  ↓
Renderer
```

## Working Spec vs Render Spec

`workingSpec` is the latest state after every accepted Patch. During streaming it may still be impossible to render because the Root or Child references are incomplete.

`renderSpec` is the most recent state that passed the Frontend Boundary Validation. If a new Patch leaves the Spec temporarily incomplete, keep the previous `renderSpec`.

## Snapshot Reconciliation

Treat the Backend's final Snapshot as the final authoritative state for that UI Part, while still validating it again on the Frontend.

```text
snapshot
  ↓ version / catalog / boundary validation
valid   → renderSpec = snapshot
invalid → keep LKG + mark part failed
```

A Snapshot synchronizes state within the same connection. It is not a protocol for automatically resuming a disconnected Stream.

## Sequence Guard

Strictly validate per-request Sequence numbers. Duplicates, reordering, and gaps are Protocol Errors by default. Do not automatically replay the entire request after some UI has already been applied.

## Part Failure Isolation

If one UI Part fails, completed Text Parts and other safe Parts may remain visible. Show a Fallback only inside the affected Part boundary.

## Security

- Runtime Schema validation for Events and Patches
- block prototype-pollution paths
- reject Components outside the Registry
- reject Actions outside the Catalog
- prohibit arbitrary HTML/script execution
- limit UI Element/State size
- revalidate server-authorized Actions on the server

## Validation

- first Patch incomplete, later promoted to a safe Spec
- invalid Patch after valid UI → preserve Last-Known-Good
- Patch result differs from Snapshot → reconcile to the valid Snapshot
- missing Snapshot / version mismatch
- EOF without a Terminal Event
- distinguish Abort from Network Failure
