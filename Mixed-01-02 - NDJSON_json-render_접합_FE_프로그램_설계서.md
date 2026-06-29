# NDJSON–json-render 접합 Frontend 프로그램 설계서

> 문서 유형: 접합 계층 설계서  
> 적용 위치: `스트리밍 프록시 Frontend`와 `json-render 기반 Frontend UI 생성 레이어` 사이  
> 기준 버전: Event Contract v1 / json-render SpecStream / Vue Renderer / RFC 6902 / NDJSON  
> 작성 목적: 브라우저에서 파싱된 NDJSON Event를 순서 있는 Message Part와 안전한 json-render Spec으로 변환한다.

---

# 1. 문서 목적

본 문서는 다음 두 Frontend 계층을 연결하는 **접합 프로그램(Adapter Layer)** 을 정의한다.

```text
ReadableStream / NDJSON Parser
    ↓ Parsed Event
NDJSON–json-render 접합 Frontend
    ├─ Contract와 Sequence 검증
    ├─ Message Part Store
    ├─ Text Delta Reducer
    ├─ UI Part Compiler
    ├─ Last-Known-Good Spec
    └─ Snapshot Reconciler
           ↓
json-render Vue UI 생성 레이어
    ├─ Registry
    ├─ StateProvider
    ├─ VisibilityProvider
    ├─ ValidationProvider
    ├─ ActionProvider
    └─ Renderer
```

기존 스트리밍 프록시 Frontend는 Byte Chunk를 UTF-8 Text로 복원하고 NDJSON 한 줄을 Event Object로 파싱한다.

기존 json-render Frontend UI 생성 레이어는 Spec을 Vue Component Tree로 렌더링한다.

본 계층은 두 모델 사이에서 다음 문제를 해결한다.

- Event Contract 및 Version 확인
- Event Sequence 누락·역순 감지
- Inline Message의 Text/UI Part 순서 복원
- Text Delta 누적
- UI Patch를 Part별 Spec으로 Compile
- 불완전 Spec으로 인한 렌더링 오류 격리
- 안전한 Spec만 Renderer에 전달
- 최종 Snapshot을 통한 Patch 결과 검증 및 복구
- Part 단위 Fallback과 전체 Stream 상태 관리
- 기존 업무 Event를 Catalog Component 기반 UI로 이전

---

# 2. 연계 대상 문서

본 문서는 다음 기존 설계서 사이의 계약을 구체화한다.

1. `스트리밍_프록시_FE_프로그램_설계서.md`
   - `fetch()`와 `ReadableStream`
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
   - `done`과 `error`

4. `잠재표준 - LLM UI BE.md`
   - Inline Mixed Stream
   - SpecStream Compile
   - Catalog Validation

---

# 3. 핵심 설계 결정

## 3.1 Event Dispatcher와 Renderer를 직접 연결하지 않는다

다음 구조를 사용하지 않는다.

```text
Event type = DealCard
    ↓
DealCard Vue Component 직접 호출
```

다음 구조를 사용한다.

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

## 3.2 Working Spec과 Render Spec을 분리한다

Streaming 중 Patch가 적용된 최신 Spec이 항상 렌더링 가능한 것은 아니다.

따라서 UI Part는 최소 두 개의 Spec 상태를 가진다.

```text
workingSpec
= 모든 정상 Patch를 적용한 최신 상태

renderSpec
= Frontend Boundary Validation을 통과한 Last-Known-Good 상태
```

Patch 적용 후 `workingSpec`이 일시적으로 불완전하면 기존 `renderSpec`을 유지한다.

## 3.3 Snapshot은 최종 권위 상태다

Backend가 전체 검증 후 보낸 `content.ui.snapshot`은 해당 UI Part의 최종 기준이다.

Frontend는 Snapshot을 다시 경계 검증한 뒤 다음과 같이 처리한다.

```text
Snapshot 유효
    ↓
renderSpec = snapshot.spec
UI Part 확정

Snapshot 무효
    ↓
기존 Last-Known-Good 유지
UI Part 실패 처리
```

## 3.4 Stream 재시도와 Snapshot 복구를 구분한다

Snapshot은 동일 연결 안에서 Patch 적용 결과를 최종 동기화하는 수단이다.

Snapshot이 있다고 해서 중간 연결 종료 후 자동 Resume가 가능한 것은 아니다.

Resume Token과 Replay API가 없는 Phase 1에서는 일부 Event를 소비한 후 자동 재시도하지 않는다.

---

# 4. 목표 구조

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

# 5. Frontend 접합 계층의 책임

본 계층은 다음을 담당한다.

- Parsed Event의 Runtime Schema 검증
- Event Contract Version 확인
- `requestId`, `messageId`, `sequence` 확인
- Event별 Payload 검증
- Part 생성과 순서 관리
- Text Delta 누적
- UI Part별 Compiler 생성 및 정리
- Patch Operation 및 Path 재검증
- Working Spec 갱신
- 렌더 가능한 Last-Known-Good Spec 관리
- Snapshot 재검증 및 최종 동기화
- Part 완료·실패 상태 반영
- 전체 완료·오류·취소 상태 반영
- UI Part 범위 Fallback
- Renderer Update Batching
- 관측성 Event 생성

본 계층은 다음을 담당하지 않는다.

- Byte Chunk와 UTF-8 경계 처리
- NDJSON 줄 분리
- LLM 호출
- Catalog Prompt 생성
- Vue Component 내부 표현
- Action의 실제 업무 로직
- 서버 권한 판정

---

# 6. 수신 Event Envelope

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

NDJSON Parser는 JSON 파싱까지만 담당한다.

그 다음 Event Contract Validator가 다음을 확인한다.

- Object 여부
- 필수 필드
- 필드 타입
- 지원 Event Version
- 알려진 Event Type
- Event Type별 Payload Schema
- 최대 Event Byte 크기

`unknown` Event Type은 기본적으로 Protocol Error로 처리한다. 하위 호환 확장을 허용하려면 별도 Capability Flag를 둔다.

---

# 7. Stream Session 상태 모델

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

## 7.1 Text Part 상태

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

## 7.2 UI Part 상태

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

Compiler 객체처럼 직렬화할 수 없는 값은 Pinia 또는 Vue Reactive State 외부의 Runtime Registry에 보관할 수 있다.

```text
Serializable Chat State
+
Request-scoped Runtime Objects
```

을 분리하는 것을 권장한다.

---

# 8. Sequence Guard

Event는 요청별 전역 Sequence를 가진다.

```ts
function assertSequence(event: StreamEvent, session: StreamSessionState) {
  if (event.sequence !== session.expectedSequence) {
    throw new StreamProtocolError("SEQUENCE_MISMATCH");
  }

  session.expectedSequence += 1;
}
```

정책:

- `sequence < expected`: 중복 또는 역순 Event로 처리
- `sequence > expected`: 누락 Event로 처리
- 둘 다 Protocol Error가 기본값
- Phase 1에서는 자동 보정하지 않는다.
- Event를 이미 UI에 반영한 뒤 전체 요청을 자동 재시도하지 않는다.

HTTP Stream 특성상 정상 연결에서는 재전송이 발생하지 않으므로 엄격한 Sequence 정책을 권장한다.

---

# 9. Event 처리 표

| Event Type | Frontend 처리 |
|---|---|
| `meta` | Version과 Catalog/Schema 호환성 확인, Session 초기화 |
| `content.part.start` | Part 생성, `partOrder`에 추가 |
| `content.text.delta` | Text Part 문자열 끝에 `delta` 추가 |
| `content.ui.patch` | UI Compiler에 Patch 적용, Working Spec 갱신 |
| `content.ui.snapshot` | 전체 검증 후 Render Spec을 Snapshot으로 확정 |
| `content.part.end` | Part 완료 또는 실패 상태 반영 |
| `status.done` | 열린 Part 확인 후 전체 완료/부분 완료 확정 |
| `status.error` | 전체 실패 확정, 열린 Part 정리 |
| `heartbeat` | `lastEventAt` 갱신, UI Part 생성 안 함 |

---

# 10. `meta` 처리

`meta`는 첫 Event여야 한다.

검증 항목:

- Event Contract Version
- Inline Mode 여부
- Catalog Version
- Schema Version
- SpecStream Version
- Snapshot Mode
- 현재 FE Registry Version과의 호환성

```ts
function handleMeta(payload: MetaPayload) {
  assert(payload.mode === "inline");
  assertSupportedCatalog(payload.catalogVersion);
  assertSupportedSchema(payload.schemaVersion);
  assertSupportedSpecStream(payload.specStreamVersion);
}
```

호환되지 않으면 UI Patch를 받기 전에 요청을 실패 처리한다.

---

# 11. Part Lifecycle 처리

## 11.1 Part 시작

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

## 11.2 Part 종료

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

## 11.3 열린 Part 규칙

- Delta 또는 Patch는 열린 Part에만 적용한다.
- 종료된 Part는 변경하지 않는다.
- 같은 `partId`를 다시 시작하지 않는다.
- `status.done` 수신 시 열린 Part가 있으면 Protocol Error가 기본값이다.

---

# 12. Text Delta 처리

```ts
function handleTextDelta(payload: TextDeltaPayload) {
  const part = requireTextPart(payload.partId);
  assert(part.status === "streaming");
  part.text += payload.delta;
}
```

렌더링 원칙:

- Delta마다 전체 Chat Tree를 재렌더링하지 않는다.
- Vue 반응성 업데이트를 짧은 Batch 또는 Animation Frame 단위로 합칠 수 있다.
- Markdown Parser 비용이 높으면 일정 길이 또는 시간 단위로 갱신한다.
- 최종 Part 종료 시 전체 Text를 한 번 확정 파싱한다.
- Markdown의 Link, Image, HTML 정책은 별도 Sanitizer를 적용한다.

---

# 13. UI Patch 처리

```text
content.ui.patch
    ↓
Payload Schema Validation
    ↓
Patch Guard
    ↓
Part Compiler 적용
    ↓
workingSpec 갱신
    ↓
Boundary Validation
    ├─ 안전 → renderSpec으로 Promote
    └─ 불완전/위험 → 기존 renderSpec 유지
```

## 13.1 Patch Guard

Backend에서 검증했더라도 Frontend 경계에서 다시 확인한다.

검증 항목:

- `op`
- `path`
- `from`
- `value`
- 허용 Root Path
- 금지 Pointer Token
- Patch 크기
- 누적 Patch 수
- 누적 Spec 크기

기본 허용 Root:

```text
/root
/elements
/state
```

금지 Token:

```text
__proto__
prototype
constructor
```

## 13.2 Compiler 적용

공식 `createSpecStreamCompiler` 또는 `applySpecPatch`를 사용한다.

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

자체 JSON Patch 구현을 만들지 않는다.

---

# 14. Boundary Validation과 Last-Known-Good

Streaming 중에는 다음과 같은 불완전 상태가 잠시 발생할 수 있다.

- `root`가 추가됐지만 해당 Element가 아직 없음
- Container의 `children`에 있는 Element가 아직 없음
- State가 추가되기 전에 Dynamic Value가 먼저 생성됨
- Parent가 Child보다 먼저 생성됨

따라서 최종 검증과 Streaming Boundary Validation을 구분한다.

## 14.1 Boundary Validation

다음 조건을 통과하면 `workingSpec`을 `renderSpec`으로 Promote할 수 있다.

- Spec Object 형식
- Root가 없으면 빈 Placeholder로 처리 가능
- Root가 있으면 Root Element가 존재
- 현재 참조되는 Element가 존재하거나 안전한 Placeholder 정책이 있음
- 모든 생성된 Component Type이 Registry/Catalog에 존재
- 현재 완성된 Props가 허용 Schema를 명백히 위반하지 않음
- Action 이름이 등록됨
- Dynamic Value가 허용 Directive만 사용
- 최대 Element·Depth·State 제한 이내

## 14.2 Promote 정책

```ts
function tryPromoteRenderSpec(part: UiPartState) {
  const result = boundaryValidator.validate(part.workingSpec);

  if (result.safeToRender) {
    part.renderSpec = cloneForRender(part.workingSpec);
    part.boundaryIssues = [];
  } else {
    part.boundaryIssues = result.issues;
    // 기존 renderSpec 유지
  }
}
```

Renderer에는 `workingSpec`이 아니라 `renderSpec`만 전달한다.

## 14.3 초기 상태

아직 한 번도 안전한 Spec이 없으면 UI Part는 Skeleton 또는 Loading Placeholder를 표시한다.

```text
renderSpec 없음 + streaming
→ Generated UI Loading Placeholder
```

---

# 15. Snapshot Reconciliation

`content.ui.snapshot` 수신 시 다음을 수행한다.

```text
Snapshot Payload Version 확인
    ↓
Spec Boundary Validation
    ↓
Catalog/Registry 호환성 확인
    ↓
Element/State 크기 제한 확인
    ↓
유효한 경우 최종 Render Spec으로 교체
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

## 15.1 Patch 결과와 Snapshot 불일치

Patch로 구성한 `workingSpec`과 Snapshot이 다를 수 있다.

정책:

- Sequence가 모두 정상이고 Snapshot이 유효하면 Snapshot을 최종 기준으로 사용한다.
- 불일치를 관측성 Event로 기록한다.
- Snapshot으로 교체한 뒤 UI Part를 확정한다.
- 불일치가 반복되면 Backend Compiler와 FE Compiler의 Version 및 구현을 점검한다.

## 15.2 Snapshot Mode

`meta.payload.snapshotMode`가 `final`이면 완료된 UI Part는 Snapshot을 받아야 한다.

`none`으로 협상된 경우 Patch 결과의 최종 `workingSpec`을 최종 검증 후 확정한다.

Phase 1 기본값은 `final`이다.

---

# 16. Renderer 연결

Message Part Store는 다음과 같이 렌더링한다.

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

UI Part는 `renderSpec`만 Renderer에 전달한다.

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

실제 Provider 중첩 방식은 사용 중인 `@json-render/vue` API에 맞춘다.

---

# 17. Component Type과 Props 연결

수신 Event는 다음과 같다.

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
          "title": "상품 A",
          "price": 12000
        }
      }
    }
  }
}
```

처리 결과는 다음과 같다.

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
props.title, props.price 전달
```

NDJSON Event Dispatcher는 `DealCard`를 알 필요가 없다.

---

# 18. Action 연결

Spec의 Action은 `props`와 분리한다.

```json
{
  "type": "Button",
  "props": {
    "label": "상세 보기"
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

Frontend 원칙:

- Catalog/Registry에 등록된 Action만 실행
- Parameter Schema 재검증
- 임의 Function String 실행 금지
- Navigation URL Allowlist
- 비동기 Action의 Loading과 Error를 UI Part 범위에서 관리
- 서버 권한이 필요한 Action은 서버에서 다시 권한 검증
- Visibility는 권한 검증 수단으로 사용하지 않음

---

# 19. 기존 Event Type 마이그레이션

기존 Dispatcher가 Event Type별로 Component를 직접 선택하는 경우 다음과 같이 변경한다.

기존:

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

변경:

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

Component 선택은 Registry로 이동한다.

| 기존 Event | 신규 처리 | Catalog Component 예시 |
|---|---|---|
| `dealCards` | UI Patch/Snapshot Compile | `DealCardList`, `DealCard` |
| `comparison` | UI Patch/Snapshot Compile | `ComparisonTable` |
| `questions` | UI Patch/Snapshot Compile | `QuestionList`, `QuestionButton` |
| `block` | UI Patch/Snapshot Compile | Spec 내부 `type`에 따라 결정 |
| `text` | Text Part Delta 누적 | 해당 없음 |

---

# 20. 완료와 오류 처리

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

이미 정상 표시된 Text와 Last-Known-Good UI는 보존할 수 있다. 전체 Chat 화면을 제거하지 않는다.

## 20.3 Part 실패

UI Part만 실패한 경우 해당 Part에만 Fallback을 표시한다.

```text
Text Part 정상
UI Part 실패
Text Part 정상
```

이면 정상 Text Part는 그대로 유지한다.

## 20.4 EOF

- `status.done` 또는 `status.error` 수신 후 EOF: 정상
- Terminal Event 없이 EOF: 비정상 종료
- 사용자 Abort 후 EOF: `cancelled`

---

# 21. 취소와 정리

사용자가 중지하거나 화면을 벗어나면 다음을 정리한다.

- `AbortController.abort()`
- Reader 취소
- Pending Text Render Batch
- UI Part Compiler
- Runtime Registry
- Timer
- Heartbeat Watchdog
- Request-scoped Store

취소는 오류 Toast를 표시하지 않는 것을 기본값으로 한다.

부분 응답 표시 여부는 제품 정책에 따르되, 취소된 UI Part는 완료된 결과로 저장하지 않는다.

---

# 22. 성능 설계

## 22.1 Patch Render Batching

Patch마다 Vue Renderer를 즉시 전체 갱신하지 않는다.

```text
여러 Patch 적용
    ↓
workingSpec 갱신
    ↓
한 Animation Frame에 한 번 Promote/Render
```

단, Text와 UI Part 순서 자체를 늦게 바꾸면 안 된다.

## 22.2 Clone 정책

Compiler가 Spec을 Mutate할 수 있으므로 Renderer에 전달할 때 Vue가 변경을 감지할 수 있는 새 Reference를 보장한다.

- 얕은 Clone이 충분한지 확인
- 필요 시 구조 공유 방식 사용
- 매 Patch마다 전체 Deep Clone은 피함
- Snapshot은 최종 Immutable State로 취급 가능

## 22.3 Message 단위 격리

- Compiler는 Message/Part 단위로 관리
- 다른 요청의 Patch가 섞이지 않도록 함
- Stable `partId`를 Vue Key로 사용
- Text Only 응답에는 json-render Runtime을 만들지 않음

---

# 23. 보안 원칙

UI Spec은 신뢰하지 않는 입력으로 취급한다.

- Event Runtime Schema 검증
- Patch Path Prototype Pollution 차단
- Registry 밖 Component 렌더링 금지
- Catalog 밖 Action 실행 금지
- 임의 HTML 또는 Script 실행 금지
- Markdown HTML 비활성화 또는 Sanitization
- 외부 Image/Link Domain 정책
- Action Parameter 크기 제한
- UI Element 및 State 크기 제한
- Dynamic Directive Allowlist
- 서버 권한 Action의 서버 측 재검증

Fallback Component가 오류 원문 또는 내부 Stack을 표시하지 않도록 한다.

---

# 24. 관측성

Frontend는 다음을 기록할 수 있다.

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

사용자 Text, Spec 전체, State 전체, Action Parameter는 기본 로그에서 제외한다.

---

# 25. 테스트 전략

## 25.1 Event Contract Test

- 필수 Envelope 필드
- 지원하지 않는 Version
- 알려지지 않은 Event Type
- Event Type별 잘못된 Payload
- 최대 Event 크기

## 25.2 Sequence Test

- 정상 순서
- 중복 Sequence
- 누락 Sequence
- 역순 Sequence
- Terminal Event 이후 Event

## 25.3 Part Lifecycle Test

- Text Only
- UI Only
- Text → UI
- UI → Text
- 여러 UI Part
- Part Start 없는 Delta
- Part End 후 Patch
- 중복 partId
- 잘못된 partIndex

## 25.4 UI Patch Test

- 정상 add/replace/remove
- 선택 Profile의 move/copy/test
- Chunk 경계와 무관한 Event 처리
- 금지 Path
- 최대 Patch 수
- Compiler 적용 오류
- Root보다 Child가 먼저 생성되는 경우
- 아직 없는 Child 참조

## 25.5 Last-Known-Good Test

- 첫 Patch는 불완전, 이후 Patch에서 유효
- 유효 Spec 이후 잘못된 Patch
- 잘못된 Working Spec에서 기존 Render Spec 유지
- Registry에 없는 Component
- Props Schema 위반

## 25.6 Snapshot Test

- Patch 결과와 동일한 Snapshot
- Patch 결과와 다른 유효 Snapshot
- 잘못된 Snapshot
- Catalog Version 불일치
- Snapshot 누락 후 Part 완료
- Snapshot Mode `none`

## 25.7 Error Test

- UI Part만 실패
- 전체 `status.error`
- Terminal Event 없는 EOF
- 사용자 Abort
- Network Disconnect
- Fallback이 다른 Part에 영향을 주지 않는지

## 25.8 통합 Test

- Browser → Nitro → Backend → LLM 전체 경로
- UTF-8 멀티바이트 Text Delta
- Text/UI 교차 순서
- 실제 Vue Registry 렌더링
- Action 실행
- 장시간 Heartbeat
- 느린 Client와 Backpressure

---

# 26. 단계별 도입

## Phase 1

- 엄격한 Event Contract v1
- 엄격한 Sequence
- Message Part Store
- Text Delta
- UI Patch
- UI Snapshot 필수
- Last-Known-Good
- Part 단위 Fallback
- Terminal Event 필수

## Phase 2

- Snapshot Mode 협상
- Snapshot Digest 비교
- Resume Token 및 Event Replay 검토
- Persisted Message Part 복원
- Devtools에서 Patch Timeline 표시

Resume 기능은 별도 프로토콜로 설계하며 단순 HTTP 재시도로 대체하지 않는다.

---

# 27. 완료 조건

- NDJSON Parser 결과가 Runtime Schema를 통과한 뒤에만 처리된다.
- Event Contract Version과 Catalog/Schema Version을 확인한다.
- 요청별 Sequence 누락과 역순을 탐지한다.
- Text와 UI Part의 원래 순서가 보존된다.
- Text Delta는 Text Part에 누적된다.
- UI Part마다 독립 SpecStream Compiler가 사용된다.
- NDJSON Event Type과 Component Type을 혼용하지 않는다.
- Patch는 Frontend에서도 Path와 크기를 검증한다.
- Working Spec과 Render Spec을 분리한다.
- 안전한 Spec만 Renderer로 Promote한다.
- 유효하지 않은 중간 Spec에서는 Last-Known-Good UI를 유지한다.
- 검증된 Snapshot을 최종 UI Spec으로 확정한다.
- Registry에 없는 Component는 실행하거나 추정하지 않는다.
- UI Part 오류는 해당 Part 범위 Fallback으로 격리한다.
- `status.done`, `status.error`, EOF, Abort를 구분한다.
- 완료 또는 취소 시 Compiler와 Reader 관련 리소스를 정리한다.

---

# 28. 공식 기준 자료

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
