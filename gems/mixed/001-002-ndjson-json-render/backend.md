# NDJSON × json-render Adapter — Backend

## Contract Gap

The Streaming Gem owns ordered Event delivery. The UI Gem owns SpecStream and Catalog Validation. Connecting them directly can blur Transport Event Types with Component Types or leave the lifecycle of incomplete Specs undefined in the transport contract.

## Event Types

```text
meta
content.part.start
content.text.delta
content.ui.patch
content.ui.snapshot
content.part.end
status.done
status.error
heartbeat
```

Transport `type` describes Event semantics. UI Component Types exist only inside Patch data such as `elements[id].type`.

## Part Lifecycle

Preserve interleaved Text and UI in Inline Output as ordered Parts.

```text
part.start(text)
text.delta*
part.end(text)
part.start(ui)
ui.patch*
ui.snapshot?
part.end(ui)
status.done
```

Part IDs are unique within a Message. No Events may be appended to a Part after it ends.

## UI Part Runtime

Each UI Part owns an independent Compiler.

```text
Patch
  ↓ Patch Guard
Shadow Compiler
  ↓
Incremental Safety Validation
  ↓
content.ui.patch
```

Validate RFC 6902 Operations and allowed Roots (`/root`, `/elements`, `/state`), and reject prototype-pollution-related paths.

## Final Snapshot

Revalidate the complete Spec when a UI Part finishes.

```text
compiled spec
  ↓ structural validation
  ↓ catalog validation
  ↓ UI policy validation
validated snapshot
```

Only a validated Snapshot may be emitted as `content.ui.snapshot`. If a UI Part fails while Text remains valid, the whole result may still complete with a `partial` outcome.

## Backpressure / Abort

Use a bounded queue between Adapter and Writer. Client Disconnect propagates cancellation through the Mixed Parser, upstream LLM, Compiler, Validation, and pending writes.

## Validation

- Event Sequence and Terminal Event
- preserve interleaved Text/UI order
- isolate Compiler state per Part
- invalid Patches never reach the Client
- final Snapshot is emitted only after complete validation
- distinguish UI Part failure from whole-stream failure
