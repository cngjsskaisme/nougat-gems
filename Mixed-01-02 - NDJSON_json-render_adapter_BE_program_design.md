# NDJSON–json-render Adapter Backend Program Design Document

> Document Type: Adapter Layer Design Document  
> Application Location: Between `json-render-based Backend UI generation layer` and `streaming proxy Backend`  
> Reference Version: Event Contract v1 / json-render SpecStream / RFC 6902 / NDJSON  
> Purpose: Transform the LLM's Inline Mixed Stream into transmittable NDJSON Events, and deliver Text and UI SpecStream as a single ordered response.

---

# 1. Document Purpose

This document defines the **Adapter Program (Adapter Layer)** that connects the following two Backend layers.

```text
json-render-based Backend UI generation layer
    ↓ Text / SpecStream Patch
NDJSON–json-render Adapter Backend
    ↓ Versioned NDJSON Event
streaming proxy Backend
    ↓ HTTP Streaming / Flush
Nitro Reverse Proxy
    ↓
Browser
```

The existing Backend UI generation layer separates plain text and json-render SpecStream Patch from LLM output and validates the Spec.

The existing streaming proxy Backend serializes structured Events into single NDJSON lines and delivers them to the browser.

This adapter layer solves the following problems between the two models.

- Transform LLM Text and UI Patch into the same Event Stream
- Separate the transmission Event `type` from the UI Component `type`
- Preserve the interleaved Text/UI order in Inline Mode
- Manage SpecStream Compiler per UI Part
- Connect Patch-level validation with final Spec validation
- Frontend synchronization and recovery through final Snapshot
- Migrate existing `dealCards`, `comparison`, `questions` Events to Catalog Components
- Distinguish Stream completion, partial failure, and total failure states

---

# 2. Related Documents

This document does not replace the following existing design documents as-is, but rather specifies the contracts between each design document.

1. `잠재표준 - LLM UI BE.md`
   - Catalog Prompt
   - Inline Mixed Stream processing
   - SpecStream Compile
   - Catalog and policy validation

2. `스트리밍_프록시_BE_프로그램_설계서.md`
   - Streaming HTTP Controller
   - NDJSON Serializer
   - Event Sequencer
   - Stream Writer and Flush
   - Backpressure and Abort

3. `잠재표준 - LLM UI FE.md`
   - Vue Registry
   - Renderer and Provider
   - SpecStream incremental rendering

4. `스트리밍_프록시_FE_프로그램_설계서.md`
   - ReadableStream
   - UTF-8 Decoder
   - NDJSON Line Parser
   - Event Dispatcher

---

# 3. Core Design Decisions

## 3.1 Separate the two kinds of `type`

The NDJSON Envelope `type` represents **the meaning of the transmission event**.

```json
{
  "type": "content.ui.patch"
}
```

The json-render Element `type` represents **the actual UI Component name**.

```json
{
  "type": "DealCard",
  "props": {
    "title": "Recommended Product"
  }
}
```

Therefore, the following structure is maintained.

```text
Event type
= content.ui.patch

payload.patch.value.type
= DealCard
```

Component names such as `DealCard`, `ComparisonTable`, and `QuestionList` are not used as the top-level NDJSON Event Type.

## 3.2 NDJSON is responsible only for the transport protocol

NDJSON is responsible only for the following.

- Event boundaries
- Event ordering
- Request and message identification
- Event type
- Payload delivery per Event
- Completion and error delivery

The NDJSON Parser or Serializer does not interpret UI Components or render Specs.

## 3.3 json-render Spec is responsible only for the UI model

Spec is responsible for the following.

- `root`
- `elements`
- Element `type`
- `props`
- `children`
- `on`
- `visible`
- `repeat`
- `state`

## 3.4 Registry is responsible only for UI implementation

The Backend does not know the Vue Component implementation.

The Backend generates and validates Specs based on Catalog Version and Schema Version. The actual Vue Component connection is handled by the Frontend Registry.

## 3.5 The full CloudEvents specification is not enforced

The Event Envelope references the common Event Metadata separation principle of CloudEvents, but does not enforce the full set of CloudEvents fields on the internal HTTP NDJSON stream.

The default contract uses a simple internal Envelope.

When external Event Bus or multi-system interoperability is needed, it can be mapped as follows.

```text
version     → specversion or dataschema extension
requestId   → correlation extension
sequence    → sequence extension
source      → source
Event type  → type
payload     → data
```

---

# 4. Target Structure

```text
Catalog
    ↓
Catalog Prompt
    ↓
LLM Gateway
    ↓ Raw text chunks
Official Mixed Stream Parser
    ├─ onText(text)
    └─ onPatch(patch)
           ↓
NDJSON–json-render Backend Adapter
    ├─ Part Lifecycle Manager
    ├─ Text Delta Batcher
    ├─ Patch Guard
    ├─ UI Part Compiler Registry
    ├─ Incremental Safety Validator
    ├─ Final Spec Validator
    ├─ Snapshot Composer
    └─ Event Composer
           ↓
Event Sequencer
    ↓
NDJSON Serializer
    ↓
Stream Writer / Flush
```

---

# 5. Responsibilities of the Backend Adapter Layer

This layer is responsible for the following.

- Receiving Mixed Stream Parser Callbacks
- Creating Text Parts and UI Parts
- Maintaining Part order
- Issuing `partId`
- Composing Text Delta Events
- Composing SpecStream Patch Events
- Validating Patch Operations and JSON Pointer Paths
- Managing Shadow Spec Compiler per UI Part
- Incremental Spec safety validation
- Final Spec structure, Catalog, and policy validation
- Generating validated final Snapshot
- Generating Part completion Events
- Generating overall completion or overall error Events
- Negotiating Event Contract Version and UI Version
- Assigning Event Sequence per request
- Generating observability information

This layer is not responsible for the following.

- Actual implementation of HTTP Socket Write and Flush
- Vue Component selection
- Vue Rendering
- Action Handler execution
- Browser State management
- Detailed implementation of the LLM Provider SDK

---

# 6. Request Capability Contract

The Frontend optionally passes the contracts it can support when making a Streaming request.

```json
{
  "requestId": "req-01",
  "conversationId": "conv-01",
  "message": "Compare products that match the conditions",
  "clientCapabilities": {
    "eventContractVersions": ["1"],
    "acceptsNdjson": true,
    "acceptsTextDelta": true,
    "ui": {
      "acceptsSpecStream": true,
      "acceptsSnapshot": true,
      "snapshotMode": "final",
      "catalogVersions": ["catalog-2026-06"],
      "schemaVersions": ["vue-flat-tree-1"],
      "registryVersion": "registry-2026-06"
    }
  }
}
```

The Backend checks the following before sending the Streaming Header.

- Existence of a supported Event Contract Version
- Catalog Version compatibility
- Schema Version compatibility
- Patch Streaming support
- Snapshot support

If there is no compatible combination, it returns HTTP 4xx without starting the Stream.

---

# 7. Common NDJSON Event Envelope

All Events use the following Envelope.

```ts
interface StreamEvent<TPayload = unknown> {
  version: "1";
  requestId: string;
  conversationId?: string;
  messageId: string;
  sequence: number;
  type: StreamEventType;
  timestamp: string;
  payload: TPayload;
}
```

## 7.1 Field Definitions

| Field | Required | Description |
|---|---:|---|
| `version` | O | NDJSON Event Contract Version |
| `requestId` | O | Single Streaming request identifier |
| `conversationId` | X | Conversation identifier |
| `messageId` | O | Assistant Message identifier generated in this response |
| `sequence` | O | Monotonically increasing sequence number per request |
| `type` | O | Transmission Event Type |
| `timestamp` | O | Event generation time, ISO-8601 |
| `payload` | O | Per-Event data |

## 7.2 Sequence Rules

- Increases from `1` per request.
- A single global Sequence is used for all Events.
- A separate Sequence is not duplicated inside the Patch.
- The order is not changed after an Event is generated.
- Events are not serialized in parallel.
- `status.done` or `status.error` is the last Sequence.

---

# 8. Event Type Contract

```ts
type StreamEventType =
  | "meta"
  | "content.part.start"
  | "content.text.delta"
  | "content.ui.patch"
  | "content.ui.snapshot"
  | "content.part.end"
  | "status.done"
  | "status.error"
  | "heartbeat";
```

## 8.1 `meta`

The first Event of the Stream.

```json
{
  "version": "1",
  "requestId": "req-01",
  "messageId": "msg-01",
  "sequence": 1,
  "type": "meta",
  "timestamp": "2026-06-29T06:00:00.000Z",
  "payload": {
    "mode": "inline",
    "catalogVersion": "catalog-2026-06",
    "schemaVersion": "vue-flat-tree-1",
    "specStreamVersion": "rfc6902-jsonl-1",
    "snapshotMode": "final"
  }
}
```

## 8.2 `content.part.start`

Declares that a Text or UI Part has started within a Message.

```json
{
  "type": "content.part.start",
  "payload": {
    "partId": "part-text-1",
    "partIndex": 0,
    "kind": "text"
  }
}
```

A UI Part starts as follows.

```json
{
  "type": "content.part.start",
  "payload": {
    "partId": "part-ui-1",
    "partIndex": 1,
    "kind": "ui",
    "catalogVersion": "catalog-2026-06",
    "schemaVersion": "vue-flat-tree-1"
  }
}
```

## 8.3 `content.text.delta`

A string fragment to append to a Text Part.

```json
{
  "type": "content.text.delta",
  "payload": {
    "partId": "part-text-1",
    "delta": "I compared based on the following conditions."
  }
}
```

## 8.4 `content.ui.patch`

A single RFC 6902 Operation to apply to a UI Part.

```json
{
  "type": "content.ui.patch",
  "payload": {
    "partId": "part-ui-1",
    "patch": {
      "op": "add",
      "path": "/elements/card-1",
      "value": {
        "type": "DealCard",
        "props": {
          "title": "Product A",
          "price": 12000
        },
        "children": []
      }
    }
  }
}
```

Only one Patch Operation is placed per Event.

## 8.5 `content.ui.snapshot`

The final Spec that has passed the Backend's full validation after the UI Part is completed.

```json
{
  "type": "content.ui.snapshot",
  "payload": {
    "partId": "part-ui-1",
    "catalogVersion": "catalog-2026-06",
    "schemaVersion": "vue-flat-tree-1",
    "patchCount": 4,
    "spec": {
      "root": "root",
      "elements": {
        "root": {
          "type": "CardList",
          "props": { "title": "Recommendation List" },
          "children": ["card-1"]
        },
        "card-1": {
          "type": "DealCard",
          "props": {
            "title": "Product A",
            "price": 12000
          },
          "children": []
        }
      },
      "state": {}
    }
  }
}
```

Snapshot principles are as follows.

- If `snapshotMode: final`, it is sent exactly once for each successfully completed UI Part.
- It is sent only after passing full structure validation and `catalog.validate()`.
- If the Patch result and Snapshot differ on the Frontend, the Snapshot is used as the final reference.
- Sending a Snapshot does not mean that errors in previous Patch Sequences are treated as normal.
- If Snapshot cost is a concern for large Specs, `none` can be negotiated via Capability, but the default is `final`.

## 8.6 `content.part.end`

Part-level completion state.

Normal completion:

```json
{
  "type": "content.part.end",
  "payload": {
    "partId": "part-ui-1",
    "kind": "ui",
    "status": "completed"
  }
}
```

Partial failure:

```json
{
  "type": "content.part.end",
  "payload": {
    "partId": "part-ui-1",
    "kind": "ui",
    "status": "failed",
    "error": {
      "code": "UI_SPEC_VALIDATION_FAILED",
      "message": "Failed to complete the UI.",
      "retryable": false
    }
  }
}
```

If only the UI Part fails and the Text Part is normal, the entire Stream does not necessarily need to be failed.

## 8.7 `status.done`

The normal termination Event for the entire request.

```json
{
  "type": "status.done",
  "payload": {
    "outcome": "success",
    "partCount": 2,
    "completedPartCount": 2,
    "failedPartCount": 0
  }
}
```

Partial success is indicated as follows.

```json
{
  "type": "status.done",
  "payload": {
    "outcome": "partial",
    "partCount": 2,
    "completedPartCount": 1,
    "failedPartCount": 1
  }
}
```

## 8.8 `status.error`

A Terminal Error where the entire request can no longer continue.

```json
{
  "type": "status.error",
  "payload": {
    "code": "UPSTREAM_LLM_FAILED",
    "message": "An error occurred while generating the response.",
    "phase": "generation",
    "retryable": true
  }
}
```

No other Events are sent after `status.error`.

## 8.9 `heartbeat`

An Event to prevent Proxy Idle Timeout.

```json
{
  "type": "heartbeat",
  "payload": {}
}
```

Heartbeat does not create a Message Part.

---

# 9. Part Lifecycle

The interleaved order of Text and UI in Inline Mode is explicitly preserved.

```text
content.part.start(text-1)
content.text.delta(text-1)
content.part.end(text-1)
content.part.start(ui-1)
content.ui.patch(ui-1)
content.ui.patch(ui-1)
content.ui.snapshot(ui-1)
content.part.end(ui-1)
content.part.start(text-2)
content.text.delta(text-2)
content.part.end(text-2)
status.done
```

Part rules are as follows.

- `partIndex` is the display order within the Assistant Message.
- A single Part has only one Kind, either `text` or `ui`.
- `partId` is unique within the Message.
- Delta or Patch is not sent before a Part starts.
- Delta or Patch is not added to the same Part after it closes.
- UI Snapshot is sent only before the UI Part closes.
- There must be no open Parts before overall completion.

---

# 10. Mixed Stream Adapter Algorithm

The official Mixed Stream Parser is used; no custom regex Parser is created.

```ts
const parser = createMixedStreamParser({
  onText: (text) => adapter.acceptText(text),
  onPatch: (patch) => adapter.acceptPatch(patch),
});

for await (const chunk of llmStream) {
  parser.push(chunk);
}

parser.flush();
await adapter.finish();
```

## 10.1 Text Processing

```text
onText(text)
    ↓
Is the current Part text?
    ├─ Yes → Add delta with the same partId
    └─ No  → Close previous Part → Start new Text Part
    ↓
Text Delta Batching
    ↓
content.text.delta
```

Text Delta does not need to create Events directly from LLM Raw Chunk boundaries.

To prevent excessive transmission of small Chunks, short Batching can be done based on one of the following.

- Maximum character count
- Maximum Byte size
- Short time Window

Batching must not change the Text/UI interleaved order. When a Patch arrives, the pending Text Buffer is flushed first.

## 10.2 Patch Processing

```text
onPatch(patch)
    ↓
Flush pending Text Buffer
    ↓
Is the current Part ui?
    ├─ Yes → Use existing UI Part
    └─ No  → Close previous Part → Start new UI Part
    ↓
Patch Guard
    ↓
Shadow Compiler apply
    ↓
Incremental Safety Validation
    ↓
Send content.ui.patch
```

## 10.3 Stream Termination Processing

```text
parser.flush()
    ↓
Are there open UI Parts?
    ├─ Yes → Final Compile and Validation
    │         ├─ Success → Snapshot → Part End completed
    │         └─ Failure → Part End failed
    └─ No
    ↓
Close open Text Parts
    ↓
status.done(success | partial)
```

---

# 11. UI Part Compiler Registry

Each UI Part has an independent Compiler.

```ts
interface UiPartRuntime {
  partId: string;
  compiler: SpecStreamCompiler<UiSpec>;
  patchCount: number;
  lastAcceptedSpec?: UiSpec;
  finalSpec?: UiSpec;
  status: "streaming" | "completed" | "failed";
}
```

Patches for multiple UI Parts are not mixed into a single Compiler.

```text
part-ui-1 → compiler-1
part-ui-2 → compiler-2
```

---

# 12. Patch Contract

json-render SpecStream uses RFC 6902 Operations.

```ts
type SpecPatch =
  | { op: "add"; path: string; value: unknown }
  | { op: "remove"; path: string }
  | { op: "replace"; path: string; value: unknown }
  | { op: "move"; path: string; from: string }
  | { op: "copy"; path: string; from: string }
  | { op: "test"; path: string; value: unknown };
```

## 12.1 Compatibility Profile

The official SpecStream supports six Operations.

The default LLM generation Profile prioritizes the following for stability.

```text
Default allowed: add, replace, remove
Optional allowed: move, copy, test
```

When `move`, `copy`, or `test` are needed, they are explicitly stated in the Catalog Prompt and Backend Policy.

## 12.2 Allowed Paths

The default allowed Roots are as follows.

```text
/root
/elements
/state
```

Only JSON Pointers below them are allowed.

The following tokens are rejected.

```text
__proto__
prototype
constructor
```

The same Path validation is applied to Operations using `from`.

## 12.3 Patch Limits

The following limits are set per request.

- Maximum Patch count
- Maximum Patch Byte size
- Maximum Path length
- Maximum JSON Pointer depth
- Maximum Value depth
- Maximum Spec Byte size
- Maximum Element count
- Maximum Children count
- Maximum State size

---

# 13. Validation Stages

Validation is divided into before Patch transmission and at UI Part completion.

## 13.1 Pre-Transmission Validation

- JSON Patch Operation Schema
- `op` allowance
- `path` and `from` allowed range
- Prohibited Pointer Tokens
- Patch size
- Per-request Patch count limit
- Shadow Compiler applicability

A Patch that fails this stage is not sent to the Frontend.

## 13.2 Incremental Safety Validation

An intermediate Spec may not yet be complete and may fail full Schema Validation.

Therefore, only the following validations are performed.

- Maximum size of the current Spec Object
- Whether the `type` of generated Elements exists in the Catalog
- Whether completed `props` do not clearly violate the Schema
- Element ID and Children reference format
- Unregistered Action names
- Dangerous strings or arbitrary code execution fields
- UI complexity limits

Incomplete references can be pending within a certain range.

## 13.3 Final Validation

At UI Part completion, all of the following are performed.

```text
compile result
    ↓
validateSpec()
    ↓
catalog.validate()
    ↓
UI Generation Policy Validation
    ↓
Version Validation
    ↓
Validated Snapshot
```

If the final validation does not pass, the Snapshot is not sent.

---

# 14. UI Patch Transmission Safety Policy

For Streaming Rendering, Patches may be delivered before final validation.

Therefore, defense layers are placed on both Backend and Frontend.

Backend:

- Send only after passing Patch Guard
- Send only after successful Shadow Compiler application
- Block Components outside the Catalog early
- Patch and Spec size limits

Frontend:

- Re-validate with the same Patch Guard
- Separate Working Spec and Render Spec
- Promote only safe states to the Renderer
- Maintain Last-Known-Good Spec on failure
- Confirm based on Snapshot after final Snapshot validation

---

# 15. Event Composer

The Event Composer does not generate business Events directly but transforms them into standard Events.

```ts
interface EventComposer {
  meta(payload: MetaPayload): StreamEvent<MetaPayload>;
  partStart(payload: PartStartPayload): StreamEvent<PartStartPayload>;
  textDelta(payload: TextDeltaPayload): StreamEvent<TextDeltaPayload>;
  uiPatch(payload: UiPatchPayload): StreamEvent<UiPatchPayload>;
  uiSnapshot(payload: UiSnapshotPayload): StreamEvent<UiSnapshotPayload>;
  partEnd(payload: PartEndPayload): StreamEvent<PartEndPayload>;
  done(payload: DonePayload): StreamEvent<DonePayload>;
  error(payload: ErrorPayload): StreamEvent<ErrorPayload>;
  heartbeat(): StreamEvent<Record<string, never>>;
}
```

Events generated by the Event Composer are passed through the Event Sequencer to the NDJSON Serializer.

---

# 16. Full NDJSON Example

The following is an example where Text and UI are generated in order.

```ndjson
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":1,"type":"meta","timestamp":"2026-06-29T06:00:00.000Z","payload":{"mode":"inline","catalogVersion":"catalog-2026-06","schemaVersion":"vue-flat-tree-1","specStreamVersion":"rfc6902-jsonl-1","snapshotMode":"final"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":2,"type":"content.part.start","timestamp":"2026-06-29T06:00:00.010Z","payload":{"partId":"part-text-1","partIndex":0,"kind":"text"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":3,"type":"content.text.delta","timestamp":"2026-06-29T06:00:00.020Z","payload":{"partId":"part-text-1","delta":"I compared based on the following conditions."}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":4,"type":"content.part.end","timestamp":"2026-06-29T06:00:00.030Z","payload":{"partId":"part-text-1","kind":"text","status":"completed"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":5,"type":"content.part.start","timestamp":"2026-06-29T06:00:00.040Z","payload":{"partId":"part-ui-1","partIndex":1,"kind":"ui","catalogVersion":"catalog-2026-06","schemaVersion":"vue-flat-tree-1"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":6,"type":"content.ui.patch","timestamp":"2026-06-29T06:00:00.050Z","payload":{"partId":"part-ui-1","patch":{"op":"add","path":"/root","value":"root"}}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":7,"type":"content.ui.patch","timestamp":"2026-06-29T06:00:00.060Z","payload":{"partId":"part-ui-1","patch":{"op":"add","path":"/elements/root","value":{"type":"ComparisonTable","props":{"title":"Product Comparison"},"children":[]}}}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":8,"type":"content.ui.snapshot","timestamp":"2026-06-29T06:00:00.070Z","payload":{"partId":"part-ui-1","catalogVersion":"catalog-2026-06","schemaVersion":"vue-flat-tree-1","patchCount":2,"spec":{"root":"root","elements":{"root":{"type":"ComparisonTable","props":{"title":"Product Comparison"},"children":[]}},"state":{}}}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":9,"type":"content.part.end","timestamp":"2026-06-29T06:00:00.080Z","payload":{"partId":"part-ui-1","kind":"ui","status":"completed"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":10,"type":"status.done","timestamp":"2026-06-29T06:00:00.090Z","payload":{"outcome":"success","partCount":2,"completedPartCount":2,"failedPartCount":0}}
```

Newlines within strings inside a single NDJSON line are represented with JSON Escape. The physical Event boundary is a single newline character.

---

# 17. Existing Event Type Migration

Assume the existing Streaming Events are as follows.

```text
text
block
block/dealCards or dealCards
block/comparison or comparison
block/questions or questions
done
error
```

They are migrated as follows.

| Existing Event | New Event | UI Catalog Representation |
|---|---|---|
| `text` | `content.text.delta` | N/A |
| `block` | `content.ui.patch` / `content.ui.snapshot` | Concretized as `elements[id].type` |
| `dealCards` | `content.ui.patch` / `content.ui.snapshot` | `DealCardList`, `DealCard` |
| `comparison` | `content.ui.patch` / `content.ui.snapshot` | `ComparisonTable` |
| `questions` | `content.ui.patch` / `content.ui.snapshot` | `QuestionList`, `QuestionButton` |
| `done` | `status.done` | N/A |
| `error` | `status.error` or failed `content.part.end` | N/A |

Business UI names are moved to Catalog Component Types, not transmission Event Types.

---

# 18. Error Policy

## 18.1 Patch Errors

```text
Patch Schema error
Path error
Patch application error
Size limit exceeded
```

Handling:

- Invalid Patches are not sent.
- The corresponding UI Part is closed as `failed`.
- If other Text Parts are valid, the overall result can be completed as `partial`.

## 18.2 Final Spec Errors

```text
validateSpec failure
catalog.validate failure
UI Policy failure
Version mismatch
```

Handling:

- Snapshot is not sent.
- The UI Part is closed as `failed`.
- The original error message and internal Stack are not exposed to the Client.

## 18.3 Overall Request Error

If the entire request cannot continue due to LLM Gateway failure, Request Context loss, Writer error, etc., it terminates after `status.error`.

## 18.4 Client Disconnect

On Client Disconnect, the following are stopped.

- Mixed Stream Parser input
- LLM Gateway Stream
- Shadow Compiler work
- Validation
- Event generation
- Pending Writer work

---

# 19. Backpressure

The adapter layer does not generate Events infinitely faster than the Stream Writer's consumption speed.

```text
LLM Chunk
    ↓
Mixed Parser
    ↓
Bounded Event Queue
    ↓
Stream Writer
```

Principles:

- The Event Queue has a per-request upper bound.
- When the Queue reaches the threshold, LLM Chunk consumption is paused.
- If the provider Stream cannot be stopped, only a limited Buffer is allowed.
- On Queue Overflow, the request is cancelled and terminated with a safe error.
- The entire Raw LLM Stream is not stored to create a Snapshot.

---

# 20. Pseudocode

```ts
async function streamIntegratedResponse(request, writer, abortSignal) {
  const negotiated = negotiateCapabilities(request.clientCapabilities);
  const session = new IntegrationSession({ request, negotiated, writer });

  await session.emitMeta();

  const parser = createMixedStreamParser({
    onText(text) {
      session.acceptText(text);
    },
    onPatch(patch) {
      session.acceptPatch(patch);
    },
  });

  try {
    for await (const chunk of llmGateway.stream(request, abortSignal)) {
      parser.push(chunk);
      await session.drainIfNeeded();
    }

    parser.flush();
    await session.finishOpenPart();
    await session.emitDone();
  } catch (error) {
    await session.fail(error);
  } finally {
    session.dispose();
  }
}
```

Core Patch processing:

```ts
async function acceptPatch(patch: SpecPatch) {
  await flushPendingText();
  const uiPart = ensureUiPart();

  patchGuard.assertAllowed(patch);
  const nextSpec = uiPart.compiler.push(JSON.stringify(patch) + "\n").result;
  incrementalValidator.assertSafe(nextSpec);

  uiPart.patchCount += 1;
  uiPart.lastAcceptedSpec = nextSpec;

  await emit("content.ui.patch", {
    partId: uiPart.partId,
    patch,
  });
}
```

Final UI Part processing:

```ts
async function finalizeUiPart(uiPart: UiPartRuntime) {
  const spec = uiPart.compiler.getResult();

  const structural = validateSpec(spec);
  const catalogResult = catalog.validate(spec);
  const policyResult = validateUiPolicy(spec);

  if (!structural.valid || !catalogResult.valid || !policyResult.valid) {
    return endPartAsFailed(uiPart, "UI_SPEC_VALIDATION_FAILED");
  }

  await emit("content.ui.snapshot", {
    partId: uiPart.partId,
    catalogVersion,
    schemaVersion,
    patchCount: uiPart.patchCount,
    spec,
  });

  await endPartAsCompleted(uiPart);
}
```

---

# 21. Security Principles

- UI Specs are not evaluated as executable code.
- LLM is not allowed to directly generate HTML, Vue Templates, or Import paths.
- Arbitrary Functions or Scripts inside `props` are not allowed.
- Actions allow only names and Parameters registered in the Catalog.
- URL or Navigation Actions apply a separate Allowlist.
- JSON Pointer Prototype Pollution paths are blocked.
- The size, depth, and repeat count of UI Specs and Patches are limited.
- User original text, full Specs, and Action Parameters are excluded from default operational logs.

---

# 22. Observability

The following are recorded per request.

```text
requestId
messageId
contract version
catalog version
schema version
part count
text part count
ui part count
text delta count
patch count
snapshot count
final element count
patch guard failure count
incremental validation result
final validation result
partial completion status
first event time
first text time
first ui patch time
stream duration
abort source
error category
```

The original Payload is not recorded in default logs.

---

# 23. Test Strategy

## 23.1 Contract Test

- All Event Envelope required fields
- Sequence monotonically increasing
- `meta` is the first Event
- `status.done` or `status.error` is the last Event
- One Event is one NDJSON line
- UTF-8 serialization

## 23.2 Mixed Stream Test

- Text Only
- UI Only
- Text → UI
- UI → Text
- Text → UI → Text → UI
- JSON Patch Line split across multiple LLM Chunks
- A single Chunk containing Text and multiple Patch Lines
- No trailing newline on the last line

## 23.3 Patch Test

- add, remove, replace
- Optional Profile move, copy, test
- Invalid Path
- Prohibited Path Token
- Patch order error
- replace/remove with no target
- Maximum Patch count exceeded

## 23.4 Validation Test

- Valid final Spec
- Unregistered Component
- Invalid Props
- Unregistered Action
- Invalid Children reference
- Maximum Element count exceeded
- Validation before Snapshot transmission

## 23.5 Failure Test

- Only UI Part fails, Text succeeds
- LLM total failure
- Client Disconnect
- Event Queue Overflow
- Snapshot generation failure
- Writer failure
- Prevent duplicate termination during Abort

---

# 24. Phased Rollout

## Phase 1

- `meta`
- `content.part.start`
- `content.text.delta`
- `content.ui.patch`
- `content.ui.snapshot`
- `content.part.end`
- `status.done`
- `status.error`
- `heartbeat`
- Snapshot Mode fixed to `final`

## Phase 2

- Client Capability negotiation
- Snapshot Mode selection
- UI Part Digest
- Reconnection and Resume Token investigation
- Optional Patch compression or Event Batching

Until Resume functionality is implemented, automatic retry after partial Stream consumption is not performed.

---

# 25. Completion Criteria

- Text and Patch from the LLM Inline Mixed Stream are separated by the official Parser.
- The interleaved order of Text and UI is preserved at the Part level.
- NDJSON Event Type and UI Component Type are separated.
- UI Patches are delivered as RFC 6902 Operations.
- Patches are sent only after Path and size validation.
- An independent Compiler is used for each UI Part.
- The final Spec passes structure, Catalog, and policy validation.
- A validated Snapshot is sent once for each UI Part in the default configuration.
- UI Part failure and overall request failure are distinguished.
- A normal Stream terminates with exactly one `status.done`.
- A total failure Stream terminates with exactly one `status.error`.
- Client Disconnect propagates to LLM and internal task cancellation.

---

# 26. Official Reference Materials

- [json-render Streaming](https://json-render.dev/docs/streaming)
- [json-render Core API](https://json-render.dev/docs/api/core)
- [json-render Generation Modes](https://json-render.dev/docs/generation-modes)
- [json-render Specs](https://json-render.dev/docs/specs)
- [json-render Catalog](https://json-render.dev/docs/catalog)
- [json-render Validation](https://json-render.dev/docs/validation)
- [RFC 6902: JSON Patch](https://www.rfc-editor.org/rfc/rfc6902.html)
- [NDJSON Specification](https://github.com/ndjson/ndjson-spec)
- [CloudEvents Specification](https://github.com/cloudevents/spec)
