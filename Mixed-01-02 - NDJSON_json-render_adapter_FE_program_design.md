# NDJSON–json-render Adapter Frontend Program Design Document

> Document Type: Adapter Layer Design Document  
> Application Location: Between `streaming proxy Frontend` and `json-render-based Frontend UI generation layer`  
> Reference Version: Event Contract v1 / json-render SpecStream / Vue Renderer / RFC 6902 / NDJSON  
> Purpose: Transform NDJSON Events parsed in the browser into ordered Message Parts and safe json-render Specs.

---

# 1. Document Purpose

This document defines the **Adapter Program (Adapter Layer)** that connects the following two Frontend layers.

```text
ReadableStream / NDJSON Parser
    ↓ Parsed Event
NDJSON–json-render Adapter Frontend
    ├─ Contract and Sequence validation
    ├─ Message Part Store
    ├─ Text Delta Reducer
    ├─ UI Part Compiler
    ├─ Last-Known-Good Spec
    └─ Snapshot Reconciler
           ↓
json-render Vue UI generation layer
    ├─ Registry
    ├─ StateProvider
    ├─ VisibilityProvider
    ├─ ValidationProvider
    ├─ ActionProvider
    └─ Renderer
```

The existing streaming proxy Frontend restores Byte Chunks to UTF-8 Text and parses each NDJSON line into an Event Object.

The existing json-render Frontend UI generation layer renders Specs into a Vue Component Tree.

This layer solves the following problems between the two models.

- Event Contract and Version verification
- Event Sequence missing and out-of-order detection
- Restoring the Text/UI Part order of Inline Messages
- Text Delta accumulation
- Compiling UI Patches into per-Part Specs
- Isolating rendering errors caused by incomplete Specs
- Delivering only safe Specs to the Renderer
- Verifying and recovering Patch results through final Snapshot
- Part-level Fallback and overall Stream state management
- Migrating existing business Events to Catalog Component-based UI

---

# 2. Related Documents

This document specifies the contracts between the following existing design documents.

1. `스트리밍_프록시_FE_프로그램_설계서.md`
   - `fetch()` and `ReadableStream`
   - Incremental Decoder
   - NDJSON Line Buffer
   - NDJSON Parser
   - Event Dispatcher

2. `잠재표준 - LLM UI FE.md`
   - Message Part Resolver
   - Spec Boundary Validator
   - Vue Registry
   - Renderer
   - State·Visibility·Validation·Action Provider

3. `스트리밍_프록시_BE_프로그램_설계서.md`
   - Event Envelope
   - Sequence
   - `done` and `error`

4. `잠재표준 - LLM UI BE.md`
   - Inline Mixed Stream
   - SpecStream Compile
   - Catalog Validation

---

# 3. Core Design Decisions

## 3.1 Event Dispatcher and Renderer are not directly connected

The following structure is not used.

```text
Event type = DealCard
    ↓
DealCard Vue Component directly called
```

The following structure is used.

```text
Event type = content.ui.patch
    ↓
payload.patch
    ↓
SpecStream Compiler
    ↓
elements[id].type = DealCard
    ↓
Vue Registry
    ↓
DealCard Vue Component
```

## 3.2 Working Spec and Render Spec are separated

The latest Spec with Patches applied during Streaming is not always renderable.

Therefore, a UI Part has at least two Spec states.

```text
workingSpec
= Latest state with all valid Patches applied

renderSpec
= Last-Known-Good state that passed Frontend Boundary Validation
```

If `workingSpec` is temporarily incomplete after a Patch is applied, the existing `renderSpec` is maintained.

## 3.3 Snapshot is the final authoritative state

The `content.ui.snapshot` sent by the Backend after full validation is the final reference for that UI Part.

The Frontend re-validates the Snapshot at the boundary and processes it as follows.

```text
Snapshot valid
    ↓
renderSpec = snapshot.spec
UI Part confirmed

Snapshot invalid
    ↓
Maintain existing Last-Known-Good
UI Part marked as failed
```

## 3.4 Stream retry and Snapshot recovery are distinguished

A Snapshot is a means to finally synchronize Patch application results within the same connection.

Having a Snapshot does not mean automatic Resume is possible after a mid-connection termination.

In Phase 1, where there is no Resume Token or Replay API, automatic retry is not performed after consuming some Events.

---

# 4. Target Structure

```text
Browser fetch()
    ↓
ReadableStream Reader
    ↓
Incremental UTF-8 Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓ Parsed unknown
Event Contract Validator
    ↓ StreamEvent
Sequence Guard
    ↓ Ordered StreamEvent
NDJSON–json-render Frontend Adapter
    ├─ Stream Session Store
    ├─ Message Part Store
    ├─ Text Delta Reducer
    ├─ UI Compiler Registry
    ├─ Patch Guard
    ├─ Boundary Validator
    ├─ Snapshot Reconciler
    └─ Terminal State Coordinator
           ↓
Chat State
    ├─ Text Part → Markdown/Text Renderer
    └─ UI Part → json-render Vue Renderer
```

---

# 5. Responsibilities of the Frontend Adapter Layer

This layer is responsible for the following.

- Runtime Schema validation of Parsed Events
- Event Contract Version verification
- `requestId`, `messageId`, `sequence` verification
- Per-Event Payload validation
- Part creation and order management
- Text Delta accumulation
- Creating and cleaning up Compiler per UI Part
- Re-validating Patch Operations and Paths
- Working Spec updates
- Managing renderable Last-Known-Good Spec
- Snapshot re-validation and final synchronization
- Reflecting Part completion and failure states
- Reflecting overall completion, error, and cancellation states
- UI Part scope Fallback
- Renderer Update Batching
- Generating observability Events

This layer is not responsible for the following.

- Byte Chunk and UTF-8 boundary handling
- NDJSON line splitting
- LLM calls
- Catalog Prompt generation
- Vue Component internal representation
- Actual business logic of Actions
- Server-side authorization decisions

---

# 6. Received Event Envelope

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

The NDJSON Parser is responsible only for JSON parsing.

Then the Event Contract Validator checks the following.

- Whether it is an Object
- Required fields
- Field types
- Supported Event Version
- Known Event Type
- Per-Event-Type Payload Schema
- Maximum Event Byte size

An `unknown` Event Type is treated as a Protocol Error by default. To allow backward-compatible extensions, a separate Capability Flag is provided.

---

# 7. Stream Session State Model

```ts
interface StreamSessionState {
  requestId: string;
  conversationId?: string;
  messageId: string;
  expectedSequence: number;
  status:
    | "connecting"
    | "streaming"
    | "completed"
    | "partial"
    | "failed"
    | "cancelled";
  meta?: MetaPayload;
  partOrder: string[];
  partsById: Record<string, MessagePartState>;
  lastEventAt?: number;
  terminalEventReceived: boolean;
}
```

```ts
type MessagePartState = TextPartState | UiPartState;
```

## 7.1 Text Part State

```ts
interface TextPartState {
  kind: "text";
  partId: string;
  partIndex: number;
  text: string;
  status: "streaming" | "completed" | "failed";
  error?: SafePartError;
}
```

## 7.2 UI Part State

```ts
interface UiPartState {
  kind: "ui";
  partId: string;
  partIndex: number;
  catalogVersion: string;
  schemaVersion: string;
  status: "streaming" | "completed" | "failed";
  compiler: SpecStreamCompiler<UiSpec>;
  workingSpec?: UiSpec;
  renderSpec?: UiSpec;
  finalSnapshot?: UiSpec;
  patchCount: number;
  lastPatch?: SpecPatch;
  boundaryIssues: BoundaryIssue[];
  error?: SafePartError;
}
```

Values that cannot be serialized, like Compiler objects, can be stored in a Runtime Registry outside Pinia or Vue Reactive State.

```text
Serializable Chat State
+
Request-scoped Runtime Objects
```

Separating these is recommended.

---

# 8. Sequence Guard

Events have a global Sequence per request.

```ts
function assertSequence(event: StreamEvent, session: StreamSessionState) {
  if (event.sequence !== session.expectedSequence) {
    throw new StreamProtocolError("SEQUENCE_MISMATCH");
  }

  session.expectedSequence += 1;
}
```

Policies:

- `sequence < expected`: Treated as duplicate or out-of-order Event
- `sequence > expected`: Treated as missing Event
- Both default to Protocol Error
- In Phase 1, no automatic correction is performed
- Automatic retry of the entire request is not performed after Events have already been reflected in the UI

Since HTTP Streams do not retransmit under normal connections, a strict Sequence policy is recommended.

---

# 9. Event Processing Table

| Event Type | Frontend Processing |
|---|---|
| `meta` | Verify Version and Catalog/Schema compatibility, initialize Session |
| `content.part.start` | Create Part, add to `partOrder` |
| `content.text.delta` | Append `delta` to the end of Text Part string |
| `content.ui.patch` | Apply Patch to UI Compiler, update Working Spec |
| `content.ui.snapshot` | Confirm Render Spec as Snapshot after full validation |
| `content.part.end` | Reflect Part completion or failure state |
| `status.done` | Confirm overall completion/partial completion after checking open Parts |
| `status.error` | Confirm total failure, clean up open Parts |
| `heartbeat` | Update `lastEventAt`, do not create UI Part |

---

# 10. `meta` Processing

`meta` must be the first Event.

Validation items:

- Event Contract Version
- Inline Mode
- Catalog Version
- Schema Version
- SpecStream Version
- Snapshot Mode
- Compatibility with the current FE Registry Version

```ts
function handleMeta(payload: MetaPayload) {
  assert(payload.mode === "inline");
  assertSupportedCatalog(payload.catalogVersion);
  assertSupportedSchema(payload.schemaVersion);
  assertSupportedSpecStream(payload.specStreamVersion);
}
```

If incompatible, the request is failed before receiving UI Patches.

---

# 11. Part Lifecycle Processing

## 11.1 Part Start

```ts
function handlePartStart(payload: PartStartPayload) {
  assert(!partsById[payload.partId]);
  assert(payload.partIndex === partOrder.length);

  if (payload.kind === "text") {
    createTextPart(payload);
  } else {
    createUiPart(payload);
  }

  partOrder.push(payload.partId);
}
```

## 11.2 Part End

```ts
function handlePartEnd(payload: PartEndPayload) {
  const part = requireOpenPart(payload.partId);

  if (part.kind === "ui" && payload.status === "completed") {
    assertSnapshotRequirementSatisfied(part);
  }

  part.status = payload.status;
  part.error = payload.error;
  disposePartRuntimeWhenSafe(part);
}
```

## 11.3 Open Part Rules

- Delta or Patch is applied only to open Parts.
- Closed Parts are not modified.
- The same `partId` is not started again.
- If there are open Parts when `status.done` is received, it is a Protocol Error by default.

---

# 12. Text Delta Processing

```ts
function handleTextDelta(payload: TextDeltaPayload) {
  const part = requireTextPart(payload.partId);
  assert(part.status === "streaming");
  part.text += payload.delta;
}
```

Rendering principles:

- The entire Chat Tree is not re-rendered for each Delta.
- Vue reactive updates can be merged in short Batches or per Animation Frame.
- If Markdown Parser cost is high, update at certain length or time intervals.
- At final Part completion, the entire Text is parsed once for confirmation.
- Markdown Link, Image, and HTML policies apply a separate Sanitizer.

---

# 13. UI Patch Processing

```text
content.ui.patch
    ↓
Payload Schema Validation
    ↓
Patch Guard
    ↓
Part Compiler apply
    ↓
workingSpec update
    ↓
Boundary Validation
    ├─ Safe → Promote to renderSpec
    └─ Incomplete/Dangerous → Maintain existing renderSpec
```

## 13.1 Patch Guard

Even if validated on the Backend, it is re-checked at the Frontend boundary.

Validation items:

- `op`
- `path`
- `from`
- `value`
- Allowed Root Path
- Prohibited Pointer Token
- Patch size
- Cumulative Patch count
- Cumulative Spec size

Default allowed Roots:

```text
/root
/elements
/state
```

Prohibited Tokens:

```text
__proto__
prototype
constructor
```

## 13.2 Compiler Application

The official `createSpecStreamCompiler` or `applySpecPatch` is used.

```ts
function applyUiPatch(part: UiPartState, patch: SpecPatch) {
  patchGuard.assertAllowed(patch);

  const line = JSON.stringify(patch) + "\n";
  const { result } = part.compiler.push(line);

  part.workingSpec = result;
  part.patchCount += 1;
  part.lastPatch = patch;

  tryPromoteRenderSpec(part);
}
```

A custom JSON Patch implementation is not created.

---

# 14. Boundary Validation and Last-Known-Good

During Streaming, the following incomplete states may temporarily occur.

- `root` has been added but the corresponding Element does not yet exist
- An Element referenced in a Container's `children` does not yet exist
- A Dynamic Value is created before State is added
- A Parent is created before its Child

Therefore, final validation and Streaming Boundary Validation are distinguished.

## 14.1 Boundary Validation

If the following conditions pass, `workingSpec` can be Promoted to `renderSpec`.

- Spec Object format
- If Root is absent, it can be treated as an empty Placeholder
- If Root is present, the Root Element exists
- Currently referenced Elements exist or a safe Placeholder policy is in place
- All generated Component Types exist in the Registry/Catalog
- Currently completed Props do not clearly violate the allowed Schema
- Action names are registered
- Dynamic Values use only allowed Directives
- Within maximum Element, Depth, and State limits

## 14.2 Promote Policy

```ts
function tryPromoteRenderSpec(part: UiPartState) {
  const result = boundaryValidator.validate(part.workingSpec);

  if (result.safeToRender) {
    part.renderSpec = cloneForRender(part.workingSpec);
    part.boundaryIssues = [];
  } else {
    part.boundaryIssues = result.issues;
    // Maintain existing renderSpec
  }
}
```

Only `renderSpec`, not `workingSpec`, is passed to the Renderer.

## 14.3 Initial State

If no safe Spec has been produced yet, the UI Part displays a Skeleton or Loading Placeholder.

```text
No renderSpec + streaming
→ Generated UI Loading Placeholder
```

---

# 15. Snapshot Reconciliation

When `content.ui.snapshot` is received, the following is performed.

```text
Check Snapshot Payload Version
    ↓
Spec Boundary Validation
    ↓
Check Catalog/Registry compatibility
    ↓
Check Element/State size limits
    ↓
If valid, replace with final Render Spec
```

```ts
function handleUiSnapshot(payload: UiSnapshotPayload) {
  const part = requireUiPart(payload.partId);

  assert(payload.catalogVersion === part.catalogVersion);
  assert(payload.schemaVersion === part.schemaVersion);

  const result = finalBoundaryValidator.validate(payload.spec);

  if (!result.valid) {
    part.status = "failed";
    part.error = safeError("INVALID_UI_SNAPSHOT");
    return;
  }

  part.finalSnapshot = payload.spec;
  part.renderSpec = payload.spec;
}
```

## 15.1 Patch Result and Snapshot Mismatch

The `workingSpec` composed by Patches may differ from the Snapshot.

Policies:

- If all Sequences are normal and the Snapshot is valid, the Snapshot is used as the final reference.
- The mismatch is recorded as an observability Event.
- After replacing with the Snapshot, the UI Part is confirmed.
- If mismatches recur, check the Version and implementation of the Backend Compiler and FE Compiler.

## 15.2 Snapshot Mode

If `meta.payload.snapshotMode` is `final`, completed UI Parts must receive a Snapshot.

If negotiated as `none`, the final `workingSpec` from Patch results is confirmed after final validation.

The Phase 1 default is `final`.

---

# 16. Renderer Connection

The Message Part Store renders as follows.

```vue
<template>
  <template v-for="partId in message.partOrder" :key="partId">
    <MessageTextPart
      v-if="message.partsById[partId].kind === 'text'"
      :part="message.partsById[partId]"
    />

    <GeneratedUiPart
      v-else
      :part="message.partsById[partId]"
    />
  </template>
</template>
```

The UI Part passes only `renderSpec` to the Renderer.

```vue
<template>
  <GeneratedUiFallback v-if="part.status === 'failed'" />

  <GeneratedUiSkeleton v-else-if="!part.renderSpec" />

  <StateProvider v-else :initial-state="part.renderSpec.state">
    <VisibilityProvider>
      <ValidationProvider>
        <ActionProvider :handlers="handlers">
          <Renderer
            :spec="part.renderSpec"
            :registry="registry"
            :loading="part.status === 'streaming'"
          />
        </ActionProvider>
      </ValidationProvider>
    </VisibilityProvider>
  </StateProvider>
</template>
```

The actual Provider nesting approach follows the `@json-render/vue` API in use.

---

# 17. Component Type and Props Connection

The received Event is as follows.

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
        }
      }
    }
  }
}
```

The processing result is as follows.

```text
Event Dispatcher
    ↓ content.ui.patch
Spec Compiler
    ↓
elements["card-1"].type = "DealCard"
    ↓
Vue Registry["DealCard"]
    ↓
DealCard Vue Component
    ↓
props.title, props.price passed
```

The NDJSON Event Dispatcher does not need to know about `DealCard`.

---

# 18. Action Connection

Spec Actions are separated from `props`.

```json
{
  "type": "Button",
  "props": {
    "label": "View Details"
  },
  "on": {
    "click": [
      {
        "action": "openProductDetail",
        "params": {
          "productId": "p-01"
        }
      }
    ]
  }
}
```

Frontend principles:

- Execute only Actions registered in the Catalog/Registry
- Re-validate Parameter Schema
- Prohibit arbitrary Function String execution
- Navigation URL Allowlist
- Manage Loading and Error for asynchronous Actions within the UI Part scope
- Actions requiring server authorization are re-validated on the server side
- Visibility is not used as an authorization check

---

# 19. Existing Event Type Migration

If the existing Dispatcher directly selects Components per Event Type, change it as follows.

Existing:

```ts
switch (event.type) {
  case "dealCards":
    renderDealCards(event.payload);
    break;
  case "comparison":
    renderComparison(event.payload);
    break;
}
```

Changed:

```ts
switch (event.type) {
  case "content.ui.patch":
    uiPartAdapter.applyPatch(event.payload);
    break;
  case "content.ui.snapshot":
    uiPartAdapter.applySnapshot(event.payload);
    break;
}
```

Component selection is moved to the Registry.

| Existing Event | New Processing | Catalog Component Example |
|---|---|---|
| `dealCards` | UI Patch/Snapshot Compile | `DealCardList`, `DealCard` |
| `comparison` | UI Patch/Snapshot Compile | `ComparisonTable` |
| `questions` | UI Patch/Snapshot Compile | `QuestionList`, `QuestionButton` |
| `block` | UI Patch/Snapshot Compile | Determined by `type` inside Spec |
| `text` | Text Part Delta accumulation | N/A |

---

# 20. Completion and Error Handling

## 20.1 `status.done`

```ts
function handleDone(payload: DonePayload) {
  assertNoOpenParts();
  assert(!session.terminalEventReceived);

  session.status =
    payload.outcome === "partial" ? "partial" : "completed";
  session.terminalEventReceived = true;

  disposeAllCompilers();
}
```

## 20.2 `status.error`

```ts
function handleError(payload: ErrorPayload) {
  session.status = "failed";
  session.terminalEventReceived = true;

  markOpenPartsFailed(payload);
  preserveCompletedParts();
  disposeAllCompilers();
}
```

Already-displayed normal Text and Last-Known-Good UI can be preserved. The entire Chat screen is not removed.

## 20.3 Part Failure

If only the UI Part fails, a Fallback is displayed only for that Part.

```text
Text Part normal
UI Part failed
Text Part normal
```

In this case, the normal Text Parts are kept as-is.

## 20.4 EOF

- EOF after receiving `status.done` or `status.error`: Normal
- EOF without a Terminal Event: Abnormal termination
- EOF after user Abort: `cancelled`

---

# 21. Cancellation and Cleanup

When the user stops or leaves the screen, the following are cleaned up.

- `AbortController.abort()`
- Reader cancellation
- Pending Text Render Batch
- UI Part Compiler
- Runtime Registry
- Timer
- Heartbeat Watchdog
- Request-scoped Store

Cancellation defaults to not showing an error Toast.

Whether to display partial responses is subject to product policy, but cancelled UI Parts are not saved as completed results.

---

# 22. Performance Design

## 22.1 Patch Render Batching

The Vue Renderer is not immediately fully updated for each Patch.

```text
Multiple Patches applied
    ↓
workingSpec updated
    ↓
Promote/Render once per Animation Frame
```

However, the Text and UI Part order itself must not be delayed.

## 22.2 Clone Policy

Since the Compiler may Mutate the Spec, ensure a new Reference that Vue can detect when passing to the Renderer.

- Check whether a shallow Clone is sufficient
- Use structural sharing if needed
- Avoid full Deep Clone on every Patch
- Snapshot can be treated as final Immutable State

## 22.3 Message-Level Isolation

- Compiler is managed per Message/Part
- Ensure Patches from different requests are not mixed
- Use stable `partId` as Vue Key
- Do not create json-render Runtime for Text-only responses

---

# 23. Security Principles

UI Specs are treated as untrusted input.

- Event Runtime Schema validation
- Patch Path Prototype Pollution blocking
- Prohibit rendering Components outside the Registry
- Prohibit executing Actions outside the Catalog
- Prohibit arbitrary HTML or Script execution
- Disable or sanitize Markdown HTML
- External Image/Link domain policy
- Action Parameter size limits
- UI Element and State size limits
- Dynamic Directive Allowlist
- Server-side re-validation of server-authorized Actions

Ensure the Fallback Component does not display the original error or internal Stack.

---

# 24. Observability

The Frontend can record the following.

```text
requestId
messageId
contract version
catalog version
schema version
first event time
first text delta time
first ui patch time
part count
text delta count
patch count
snapshot count
last sequence
sequence mismatch
patch rejection
working-to-render promotion count
last-known-good retention count
snapshot mismatch count
registry lookup failure
part fallback count
stream completion status
abort reason
stream duration
```

User Text, full Spec, full State, and Action Parameters are excluded from default logs.

---

# 25. Test Strategy

## 25.1 Event Contract Test

- Required Envelope fields
- Unsupported Version
- Unknown Event Type
- Invalid Payload per Event Type
- Maximum Event size

## 25.2 Sequence Test

- Normal order
- Duplicate Sequence
- Missing Sequence
- Out-of-order Sequence
- Event after Terminal Event

## 25.3 Part Lifecycle Test

- Text Only
- UI Only
- Text → UI
- UI → Text
- Multiple UI Parts
- Delta without Part Start
- Patch after Part End
- Duplicate partId
- Invalid partIndex

## 25.4 UI Patch Test

- Normal add/replace/remove
- Optional Profile move/copy/test
- Event processing independent of Chunk boundaries
- Prohibited Path
- Maximum Patch count
- Compiler application error
- Child created before Root
- Reference to not-yet-existing Child

## 25.5 Last-Known-Good Test

- First Patch is incomplete, valid in subsequent Patches
- Invalid Patch after valid Spec
- Maintain existing Render Spec on invalid Working Spec
- Component not in Registry
- Props Schema violation

## 25.6 Snapshot Test

- Snapshot identical to Patch result
- Valid Snapshot different from Patch result
- Invalid Snapshot
- Catalog Version mismatch
- Part completion without Snapshot
- Snapshot Mode `none`

## 25.7 Error Test

- Only UI Part fails
- Overall `status.error`
- EOF without Terminal Event
- User Abort
- Network Disconnect
- Fallback does not affect other Parts

## 25.8 Integration Test

- Browser → Nitro → Backend → LLM full path
- UTF-8 multibyte Text Delta
- Text/UI interleaved order
- Actual Vue Registry rendering
- Action execution
- Long-duration Heartbeat
- Slow Client and Backpressure

---

# 26. Phased Rollout

## Phase 1

- Strict Event Contract v1
- Strict Sequence
- Message Part Store
- Text Delta
- UI Patch
- UI Snapshot required
- Last-Known-Good
- Part-level Fallback
- Terminal Event required

## Phase 2

- Snapshot Mode negotiation
- Snapshot Digest comparison
- Resume Token and Event Replay investigation
- Persisted Message Part restoration
- Display Patch Timeline in Devtools

Resume functionality is designed as a separate protocol and is not replaced by simple HTTP retry.

---

# 27. Completion Criteria

- NDJSON Parser results are processed only after passing Runtime Schema.
- Event Contract Version and Catalog/Schema Version are verified.
- Per-request Sequence missing and out-of-order are detected.
- The original order of Text and UI Parts is preserved.
- Text Delta is accumulated in Text Parts.
- An independent SpecStream Compiler is used for each UI Part.
- NDJSON Event Type and Component Type are not mixed.
- Patches are validated for Path and size on the Frontend as well.
- Working Spec and Render Spec are separated.
- Only safe Specs are Promoted to the Renderer.
- Last-Known-Good UI is maintained for invalid intermediate Specs.
- Validated Snapshot is confirmed as the final UI Spec.
- Components not in the Registry are not executed or inferred.
- UI Part errors are isolated with Part-scope Fallback.
- `status.done`, `status.error`, EOF, and Abort are distinguished.
- On completion or cancellation, Compiler and Reader-related resources are cleaned up.

---

# 28. Official Reference Materials

- [json-render Streaming](https://json-render.dev/docs/streaming)
- [json-render Core API](https://json-render.dev/docs/api/core)
- [json-render Specs](https://json-render.dev/docs/specs)
- [json-render Registry](https://json-render.dev/docs/registry)
- [json-render Renderers](https://json-render.dev/docs/renderers)
- [json-render Vue API](https://json-render.dev/docs/api/vue)
- [json-render Validation](https://json-render.dev/docs/validation)
- [RFC 6902: JSON Patch](https://www.rfc-editor.org/rfc/rfc6902.html)
- [NDJSON Specification](https://github.com/ndjson/ndjson-spec)
- [CloudEvents Specification](https://github.com/cloudevents/spec)
