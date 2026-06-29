# NDJSON–json-render 접합 Backend 프로그램 설계서

> 문서 유형: 접합 계층 설계서  
> 적용 위치: `json-render 기반 Backend UI 생성 레이어`와 `스트리밍 프록시 Backend` 사이  
> 기준 버전: Event Contract v1 / json-render SpecStream / RFC 6902 / NDJSON  
> 작성 목적: LLM의 Inline Mixed Stream을 전송 가능한 NDJSON Event로 변환하고, Text와 UI SpecStream을 하나의 순서 있는 응답으로 전달한다.

---

# 1. 문서 목적

본 문서는 다음 두 Backend 계층을 연결하는 **접합 프로그램(Adapter Layer)** 을 정의한다.

```text
json-render 기반 Backend UI 생성 레이어
    ↓ Text / SpecStream Patch
NDJSON–json-render 접합 Backend
    ↓ Versioned NDJSON Event
스트리밍 프록시 Backend
    ↓ HTTP Streaming / Flush
Nitro Reverse Proxy
    ↓
Browser
```

기존 Backend UI 생성 레이어는 LLM 출력에서 일반 텍스트와 json-render SpecStream Patch를 분리하고 Spec을 검증한다.

기존 스트리밍 프록시 Backend는 구조화 Event를 NDJSON 한 줄로 직렬화하여 브라우저로 전달한다.

본 접합 계층은 두 모델 사이에서 다음 문제를 해결한다.

- LLM Text와 UI Patch를 동일한 Event Stream으로 변환
- 전송 Event의 `type`과 UI Component의 `type` 분리
- Inline Mode의 Text/UI 교차 순서 보존
- UI Part별 SpecStream Compiler 관리
- Patch 수준 검증과 최종 Spec 검증 연결
- 최종 Snapshot을 통한 Frontend 동기화 및 복구
- 기존 `dealCards`, `comparison`, `questions` Event를 Catalog Component로 이전
- Stream 완료·부분 실패·전체 실패 상태 구분

---

# 2. 연계 대상 문서

본 문서는 다음 기존 설계서를 변경 없이 대체하는 문서가 아니라, 각 설계서 사이의 계약을 구체화하는 문서다.

1. `잠재표준 - LLM UI BE.md`
   - Catalog Prompt
   - Inline Mixed Stream 처리
   - SpecStream Compile
   - Catalog 및 정책 검증

2. `스트리밍_프록시_BE_프로그램_설계서.md`
   - Streaming HTTP Controller
   - NDJSON Serializer
   - Event Sequencer
   - Stream Writer와 Flush
   - Backpressure와 Abort

3. `잠재표준 - LLM UI FE.md`
   - Vue Registry
   - Renderer와 Provider
   - SpecStream 점진적 렌더링

4. `스트리밍_프록시_FE_프로그램_설계서.md`
   - ReadableStream
   - UTF-8 Decoder
   - NDJSON Line Parser
   - Event Dispatcher

---

# 3. 핵심 설계 결정

## 3.1 두 종류의 `type`을 분리한다

NDJSON Envelope의 `type`은 **전송 이벤트의 의미**를 나타낸다.

```json
{
  "type": "content.ui.patch"
}
```

json-render Element의 `type`은 **실제 UI Component 이름**을 나타낸다.

```json
{
  "type": "DealCard",
  "props": {
    "title": "추천 상품"
  }
}
```

따라서 다음 구조를 유지한다.

```text
Event type
= content.ui.patch

payload.patch.value.type
= DealCard
```

`DealCard`, `ComparisonTable`, `QuestionList`와 같은 Component 이름을 NDJSON 최상위 Event Type으로 사용하지 않는다.

## 3.2 NDJSON은 전송 프로토콜만 담당한다

NDJSON은 다음만 담당한다.

- Event 경계
- Event 순서
- 요청 및 메시지 식별
- Event 종류
- Event별 Payload 전달
- 완료와 오류 전달

NDJSON Parser 또는 Serializer가 UI Component를 해석하거나 Spec을 렌더링하지 않는다.

## 3.3 json-render Spec은 UI 모델만 담당한다

Spec은 다음을 담당한다.

- `root`
- `elements`
- Element의 `type`
- `props`
- `children`
- `on`
- `visible`
- `repeat`
- `state`

## 3.4 Registry는 UI 구현만 담당한다

Backend는 Vue Component 구현을 알지 않는다.

Backend는 Catalog Version과 Schema Version을 기준으로 Spec을 생성하고 검증한다. 실제 Vue Component 연결은 Frontend Registry가 담당한다.

## 3.5 CloudEvents 전체 규격을 강제하지 않는다

Event Envelope는 CloudEvents의 공통 Event Metadata 분리 원칙을 참고하지만, 내부 HTTP NDJSON 스트림에 CloudEvents 필드 전체를 강제하지 않는다.

기본 계약은 단순한 내부 Envelope를 사용한다.

외부 Event Bus 또는 다중 시스템 상호운용이 필요해지는 경우 다음과 같이 매핑할 수 있다.

```text
version     → specversion 또는 dataschema 확장
requestId   → correlation extension
sequence    → sequence extension
source      → source
Event type  → type
payload     → data
```

---

# 4. 목표 구조

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

# 5. Backend 접합 계층의 책임

본 계층은 다음을 담당한다.

- Mixed Stream Parser Callback 수신
- Text Part와 UI Part 생성
- Part 순서 유지
- `partId` 발급
- Text Delta Event 구성
- SpecStream Patch Event 구성
- Patch Operation 및 JSON Pointer Path 검증
- UI Part별 Shadow Spec Compiler 관리
- 점진적 Spec 안전성 검증
- 최종 Spec 구조·Catalog·정책 검증
- 검증된 최종 Snapshot 생성
- Part 완료 Event 생성
- 전체 완료 또는 전체 오류 Event 생성
- Event Contract Version 및 UI Version 협상
- 요청별 Event Sequence 부여
- 관측 정보 생성

본 계층은 다음을 담당하지 않는다.

- HTTP Socket Write와 Flush의 실제 구현
- Vue Component 선택
- Vue Rendering
- Action Handler 실행
- Browser State 관리
- LLM Provider SDK의 세부 구현

---

# 6. 요청 Capability 계약

Frontend는 Streaming 요청 시 지원 가능한 계약을 선택적으로 전달한다.

```json
{
  "requestId": "req-01",
  "conversationId": "conv-01",
  "message": "조건에 맞는 상품을 비교해줘",
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

Backend는 Streaming Header를 전송하기 전에 다음을 확인한다.

- 지원 가능한 Event Contract Version 존재
- Catalog Version 호환
- Schema Version 호환
- Patch Streaming 지원 여부
- Snapshot 지원 여부

호환 가능한 조합이 없으면 Stream을 시작하지 않고 HTTP 4xx로 반환한다.

---

# 7. 공통 NDJSON Event Envelope

모든 Event는 다음 Envelope를 사용한다.

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

## 7.1 필드 정의

| 필드 | 필수 | 설명 |
|---|---:|---|
| `version` | O | NDJSON Event Contract Version |
| `requestId` | O | 한 번의 Streaming 요청 식별자 |
| `conversationId` | X | 대화 식별자 |
| `messageId` | O | 이번 응답에서 생성되는 Assistant Message 식별자 |
| `sequence` | O | 요청 단위 단조 증가 순번 |
| `type` | O | 전송 Event Type |
| `timestamp` | O | Event 생성 시각, ISO-8601 |
| `payload` | O | Event별 데이터 |

## 7.2 Sequence 규칙

- 요청별로 `1`부터 증가한다.
- 모든 Event에 하나의 전역 Sequence를 사용한다.
- Patch 내부에 별도 Sequence를 중복 저장하지 않는다.
- Event를 생성한 뒤 순서를 변경하지 않는다.
- Event를 병렬로 직렬화하지 않는다.
- `status.done` 또는 `status.error`가 마지막 Sequence다.

---

# 8. Event Type 계약

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

Stream의 첫 번째 Event다.

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

Message 안에서 Text 또는 UI Part가 시작됨을 명시한다.

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

UI Part는 다음과 같이 시작한다.

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

Text Part에 추가할 문자열 조각이다.

```json
{
  "type": "content.text.delta",
  "payload": {
    "partId": "part-text-1",
    "delta": "아래 조건을 기준으로 비교했습니다."
  }
}
```

## 8.4 `content.ui.patch`

UI Part에 적용할 RFC 6902 Operation 한 개다.

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
        },
        "children": []
      }
    }
  }
}
```

한 Event에는 Patch Operation 하나만 넣는다.

## 8.5 `content.ui.snapshot`

UI Part가 완료된 후 Backend의 전체 검증을 통과한 최종 Spec이다.

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
          "props": { "title": "추천 목록" },
          "children": ["card-1"]
        },
        "card-1": {
          "type": "DealCard",
          "props": {
            "title": "상품 A",
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

Snapshot 원칙은 다음과 같다.

- `snapshotMode: final`이면 정상 UI Part마다 정확히 한 번 전송한다.
- 전체 구조 검증과 `catalog.validate()` 통과 후에만 전송한다.
- Frontend에서 Patch 결과와 Snapshot이 다르면 Snapshot을 최종 기준으로 사용한다.
- Snapshot을 전송했다고 이전 Patch Sequence의 오류를 정상으로 간주하지 않는다.
- 대규모 Spec에서 Snapshot 비용이 문제가 되면 Capability로 `none`을 협상할 수 있으나 기본값은 `final`이다.

## 8.6 `content.part.end`

Part 단위 완료 상태다.

정상 완료:

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

부분 실패:

```json
{
  "type": "content.part.end",
  "payload": {
    "partId": "part-ui-1",
    "kind": "ui",
    "status": "failed",
    "error": {
      "code": "UI_SPEC_VALIDATION_FAILED",
      "message": "UI를 완성하지 못했습니다.",
      "retryable": false
    }
  }
}
```

UI Part만 실패하고 Text Part가 정상인 경우 전체 Stream을 반드시 실패시킬 필요는 없다.

## 8.7 `status.done`

전체 요청의 정상적인 종료 Event다.

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

부분 성공은 다음과 같이 표시한다.

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

요청 전체를 더 이상 계속할 수 없는 Terminal Error다.

```json
{
  "type": "status.error",
  "payload": {
    "code": "UPSTREAM_LLM_FAILED",
    "message": "응답 생성 중 오류가 발생했습니다.",
    "phase": "generation",
    "retryable": true
  }
}
```

`status.error` 이후 다른 Event를 전송하지 않는다.

## 8.9 `heartbeat`

Proxy Idle Timeout 방지용 Event다.

```json
{
  "type": "heartbeat",
  "payload": {}
}
```

Heartbeat는 Message Part를 만들지 않는다.

---

# 9. Part Lifecycle

Inline Mode의 Text와 UI 교차 순서를 명시적으로 보존한다.

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

Part 규칙은 다음과 같다.

- `partIndex`는 Assistant Message 안의 표시 순서다.
- 하나의 Part는 `text` 또는 `ui` 중 하나의 Kind만 가진다.
- `partId`는 Message 안에서 유일하다.
- Part 시작 전에 Delta 또는 Patch를 전송하지 않는다.
- Part 종료 후 같은 Part에 Delta 또는 Patch를 추가하지 않는다.
- UI Snapshot은 UI Part 종료 전에만 전송한다.
- 전체 완료 전에 열린 Part가 없어야 한다.

---

# 10. Mixed Stream 접합 알고리즘

공식 Mixed Stream Parser를 사용하고 자체 정규식 Parser를 만들지 않는다.

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

## 10.1 Text 처리

```text
onText(text)
    ↓
현재 Part가 text인가?
    ├─ Yes → 같은 partId로 delta 추가
    └─ No  → 이전 Part 종료 → 새 Text Part 시작
    ↓
Text Delta Batching
    ↓
content.text.delta
```

Text Delta는 LLM Raw Chunk 경계를 그대로 Event로 만들 필요가 없다.

작은 Chunk 과다 전송을 방지하기 위해 다음 중 하나를 기준으로 짧게 Batch할 수 있다.

- 최대 문자 수
- 최대 Byte 수
- 짧은 시간 Window

Batching은 Text/UI 교차 순서를 변경해서는 안 된다. Patch가 도착하면 대기 중인 Text Buffer를 먼저 Flush한다.

## 10.2 Patch 처리

```text
onPatch(patch)
    ↓
대기 중 Text Buffer Flush
    ↓
현재 Part가 ui인가?
    ├─ Yes → 기존 UI Part 사용
    └─ No  → 이전 Part 종료 → 새 UI Part 시작
    ↓
Patch Guard
    ↓
Shadow Compiler 적용
    ↓
Incremental Safety Validation
    ↓
content.ui.patch 전송
```

## 10.3 Stream 종료 처리

```text
parser.flush()
    ↓
열린 UI Part 존재?
    ├─ Yes → 최종 Compile 및 Validation
    │         ├─ 성공 → Snapshot → Part End completed
    │         └─ 실패 → Part End failed
    └─ No
    ↓
열린 Text Part 종료
    ↓
status.done(success | partial)
```

---

# 11. UI Part Compiler Registry

각 UI Part는 독립된 Compiler를 가진다.

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

여러 UI Part의 Patch를 하나의 Compiler에 섞지 않는다.

```text
part-ui-1 → compiler-1
part-ui-2 → compiler-2
```

---

# 12. Patch Contract

json-render SpecStream은 RFC 6902 Operation을 사용한다.

```ts
type SpecPatch =
  | { op: "add"; path: string; value: unknown }
  | { op: "remove"; path: string }
  | { op: "replace"; path: string; value: unknown }
  | { op: "move"; path: string; from: string }
  | { op: "copy"; path: string; from: string }
  | { op: "test"; path: string; value: unknown };
```

## 12.1 호환 Profile

공식 SpecStream은 여섯 Operation을 지원한다.

LLM 생성 기본 Profile은 안정성을 위해 다음을 우선한다.

```text
기본 허용: add, replace, remove
선택 허용: move, copy, test
```

`move`, `copy`, `test`가 필요한 경우 Catalog Prompt와 Backend Policy에 명시한다.

## 12.2 허용 Path

기본 허용 Root는 다음과 같다.

```text
/root
/elements
/state
```

그 하위 JSON Pointer만 허용한다.

다음과 같은 토큰은 거부한다.

```text
__proto__
prototype
constructor
```

`from`을 사용하는 Operation에도 동일한 Path 검증을 적용한다.

## 12.3 Patch 제한

요청별로 다음 제한을 둔다.

- 최대 Patch 수
- 최대 Patch Byte 크기
- 최대 Path 길이
- 최대 JSON Pointer 깊이
- 최대 Value 깊이
- 최대 Spec Byte 크기
- 최대 Element 수
- 최대 Children 수
- 최대 State 크기

---

# 13. 검증 단계

검증은 Patch 전송 전과 UI Part 완료 시점으로 나눈다.

## 13.1 Patch 전송 전 검증

- JSON Patch Operation Schema
- `op` 허용 여부
- `path` 및 `from` 허용 범위
- 금지 Pointer Token
- Patch 크기
- 요청별 Patch 수 제한
- Shadow Compiler 적용 가능 여부

이 단계에 실패한 Patch는 Frontend로 전송하지 않는다.

## 13.2 Incremental Safety Validation

중간 Spec은 아직 완성되지 않아 전체 Schema Validation에 실패할 수 있다.

따라서 다음 검증만 수행한다.

- 현재 Spec Object의 최대 크기
- 생성된 Element의 `type`이 Catalog에 존재하는지
- 완성된 `props`가 명백히 Schema를 위반하지 않는지
- Element ID와 Children 참조 형식
- 허용되지 않은 Action 이름
- 위험한 문자열 또는 임의 실행 코드 필드
- UI 복잡도 제한

불완전 참조는 일정 범위에서 보류할 수 있다.

## 13.3 최종 검증

UI Part 종료 시 다음을 모두 수행한다.

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

최종 검증을 통과하지 못하면 Snapshot을 전송하지 않는다.

---

# 14. UI Patch 전송 안전 정책

Streaming Rendering을 위해 Patch는 최종 검증 전에도 전달될 수 있다.

따라서 Backend와 Frontend 양쪽에 방어 계층을 둔다.

Backend:

- Patch Guard 통과 후에만 전송
- Shadow Compiler 적용 성공 후에만 전송
- Catalog 밖 Component를 조기에 차단
- Patch 및 Spec 크기 제한

Frontend:

- 동일 Patch Guard 재검증
- Working Spec과 Render Spec 분리
- 안전한 상태만 Renderer에 Promote
- 실패 시 Last-Known-Good Spec 유지
- 최종 Snapshot 검증 후 Snapshot을 기준으로 확정

---

# 15. Event Composer

Event Composer는 업무 Event를 직접 생성하지 않고 표준 Event로 변환한다.

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

Event Composer가 생성한 Event는 Event Sequencer를 거쳐 NDJSON Serializer로 전달한다.

---

# 16. 전체 NDJSON 예시

다음은 Text와 UI가 순서대로 생성되는 예다.

```ndjson
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":1,"type":"meta","timestamp":"2026-06-29T06:00:00.000Z","payload":{"mode":"inline","catalogVersion":"catalog-2026-06","schemaVersion":"vue-flat-tree-1","specStreamVersion":"rfc6902-jsonl-1","snapshotMode":"final"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":2,"type":"content.part.start","timestamp":"2026-06-29T06:00:00.010Z","payload":{"partId":"part-text-1","partIndex":0,"kind":"text"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":3,"type":"content.text.delta","timestamp":"2026-06-29T06:00:00.020Z","payload":{"partId":"part-text-1","delta":"아래 조건을 기준으로 비교했습니다."}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":4,"type":"content.part.end","timestamp":"2026-06-29T06:00:00.030Z","payload":{"partId":"part-text-1","kind":"text","status":"completed"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":5,"type":"content.part.start","timestamp":"2026-06-29T06:00:00.040Z","payload":{"partId":"part-ui-1","partIndex":1,"kind":"ui","catalogVersion":"catalog-2026-06","schemaVersion":"vue-flat-tree-1"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":6,"type":"content.ui.patch","timestamp":"2026-06-29T06:00:00.050Z","payload":{"partId":"part-ui-1","patch":{"op":"add","path":"/root","value":"root"}}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":7,"type":"content.ui.patch","timestamp":"2026-06-29T06:00:00.060Z","payload":{"partId":"part-ui-1","patch":{"op":"add","path":"/elements/root","value":{"type":"ComparisonTable","props":{"title":"상품 비교"},"children":[]}}}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":8,"type":"content.ui.snapshot","timestamp":"2026-06-29T06:00:00.070Z","payload":{"partId":"part-ui-1","catalogVersion":"catalog-2026-06","schemaVersion":"vue-flat-tree-1","patchCount":2,"spec":{"root":"root","elements":{"root":{"type":"ComparisonTable","props":{"title":"상품 비교"},"children":[]}},"state":{}}}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":9,"type":"content.part.end","timestamp":"2026-06-29T06:00:00.080Z","payload":{"partId":"part-ui-1","kind":"ui","status":"completed"}}
{"version":"1","requestId":"req-01","conversationId":"conv-01","messageId":"msg-01","sequence":10,"type":"status.done","timestamp":"2026-06-29T06:00:00.090Z","payload":{"outcome":"success","partCount":2,"completedPartCount":2,"failedPartCount":0}}
```

NDJSON 한 줄 안의 문자열 줄바꿈은 JSON Escape로 표현한다. 물리적인 Event 경계는 개행 문자 하나다.

---

# 17. 기존 Event Type 마이그레이션

기존 Streaming Event가 다음과 같다고 가정한다.

```text
text
block
block/dealCards 또는 dealCards
block/comparison 또는 comparison
block/questions 또는 questions
done
error
```

다음과 같이 이전한다.

| 기존 Event | 신규 Event | UI Catalog 표현 |
|---|---|---|
| `text` | `content.text.delta` | 해당 없음 |
| `block` | `content.ui.patch` / `content.ui.snapshot` | `elements[id].type`으로 구체화 |
| `dealCards` | `content.ui.patch` / `content.ui.snapshot` | `DealCardList`, `DealCard` |
| `comparison` | `content.ui.patch` / `content.ui.snapshot` | `ComparisonTable` |
| `questions` | `content.ui.patch` / `content.ui.snapshot` | `QuestionList`, `QuestionButton` |
| `done` | `status.done` | 해당 없음 |
| `error` | `status.error` 또는 실패한 `content.part.end` | 해당 없음 |

업무 UI 이름은 전송 Event Type이 아니라 Catalog Component Type으로 이동한다.

---

# 18. 오류 정책

## 18.1 Patch 오류

```text
Patch Schema 오류
Path 오류
Patch 적용 오류
크기 제한 초과
```

처리:

- 잘못된 Patch를 전송하지 않는다.
- 해당 UI Part를 `failed`로 종료한다.
- 다른 Text Part가 유효하면 전체 결과를 `partial`로 완료할 수 있다.

## 18.2 최종 Spec 오류

```text
validateSpec 실패
catalog.validate 실패
UI Policy 실패
Version 불일치
```

처리:

- Snapshot을 전송하지 않는다.
- UI Part를 `failed`로 종료한다.
- 오류 원문과 내부 Stack은 Client에 노출하지 않는다.

## 18.3 전체 요청 오류

LLM Gateway 실패, Request Context 손실, Writer 오류 등 요청 전체를 계속할 수 없는 경우 `status.error` 후 종료한다.

## 18.4 Client Disconnect

Client Disconnect 시 다음을 중단한다.

- Mixed Stream Parser 입력
- LLM Gateway Stream
- Shadow Compiler 작업
- Validation
- Event 생성
- Pending Writer 작업

---

# 19. Backpressure

접합 계층은 Stream Writer의 소비 속도보다 Event를 무한히 빠르게 생성하지 않는다.

```text
LLM Chunk
    ↓
Mixed Parser
    ↓
Bounded Event Queue
    ↓
Stream Writer
```

원칙:

- Event Queue는 요청별 상한을 가진다.
- Queue가 임계치에 도달하면 LLM Chunk 소비를 일시 중단한다.
- 공급자 Stream을 멈출 수 없으면 제한된 Buffer만 허용한다.
- Queue Overflow 시 요청을 취소하고 안전한 오류로 종료한다.
- Snapshot을 만들기 위해 전체 Raw LLM Stream을 저장하지 않는다.

---

# 20. 의사 코드

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

Patch 처리 핵심:

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

최종 UI Part 처리:

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

# 21. 보안 원칙

- UI Spec을 실행 코드로 평가하지 않는다.
- HTML, Vue Template, Import 경로를 LLM이 직접 생성하지 못하게 한다.
- `props` 내부의 임의 Function 또는 Script를 허용하지 않는다.
- Action은 Catalog에 등록된 이름과 Parameter만 허용한다.
- URL 또는 Navigation Action은 별도 Allowlist를 적용한다.
- JSON Pointer Prototype Pollution 경로를 차단한다.
- UI Spec 및 Patch의 크기·깊이·반복 수를 제한한다.
- 사용자 원문, 전체 Spec, Action Parameter는 기본 운영 로그에서 제외한다.

---

# 22. 관측성

요청별로 다음을 기록한다.

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
partial completion 여부
first event time
first text time
first ui patch time
stream duration
abort source
error category
```

Payload 원문은 기본 로그에 기록하지 않는다.

---

# 23. 테스트 전략

## 23.1 Contract Test

- 모든 Event Envelope 필수 필드
- Sequence 단조 증가
- `meta`가 첫 Event인지
- `status.done` 또는 `status.error`가 마지막 Event인지
- 한 Event가 한 줄 NDJSON인지
- UTF-8 직렬화

## 23.2 Mixed Stream Test

- Text Only
- UI Only
- Text → UI
- UI → Text
- Text → UI → Text → UI
- JSON Patch Line이 여러 LLM Chunk로 나뉜 경우
- 한 Chunk에 Text와 여러 Patch Line이 포함된 경우
- 마지막 줄 개행이 없는 경우

## 23.3 Patch Test

- add, remove, replace
- 선택 Profile의 move, copy, test
- 잘못된 Path
- 금지 Path Token
- Patch 순서 오류
- 적용 대상이 없는 replace/remove
- 최대 Patch 수 초과

## 23.4 Validation Test

- 정상 최종 Spec
- 미등록 Component
- 잘못된 Props
- 미등록 Action
- 잘못된 Children 참조
- 최대 Element 수 초과
- Snapshot 전송 전 검증 확인

## 23.5 Failure Test

- UI Part만 실패하고 Text는 성공
- LLM 전체 실패
- Client Disconnect
- Event Queue Overflow
- Snapshot 생성 실패
- Writer 실패
- Abort 중 중복 종료 방지

---

# 24. 단계별 도입

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
- Snapshot Mode `final` 고정

## Phase 2

- Client Capability 협상
- Snapshot Mode 선택
- UI Part별 Digest
- 재연결 및 Resume Token 검토
- 선택적 Patch 압축 또는 Event Batching

Resume 기능을 구현하기 전에는 부분 Stream 소비 후 자동 재시도를 하지 않는다.

---

# 25. 완료 조건

- LLM Inline Mixed Stream의 Text와 Patch가 공식 Parser로 분리된다.
- Text와 UI의 교차 순서가 Part 단위로 보존된다.
- NDJSON Event Type과 UI Component Type이 분리된다.
- UI Patch는 RFC 6902 Operation으로 전달된다.
- Patch는 Path 및 크기 검증 후에만 전송된다.
- UI Part마다 독립 Compiler가 사용된다.
- 최종 Spec은 구조, Catalog, 정책 검증을 모두 통과한다.
- 기본 설정에서 검증된 Snapshot이 UI Part마다 한 번 전송된다.
- UI Part 실패와 전체 요청 실패가 구분된다.
- 정상 Stream은 정확히 한 개의 `status.done`으로 종료된다.
- 전체 실패 Stream은 정확히 한 개의 `status.error`로 종료된다.
- Client Disconnect가 LLM과 내부 작업 취소로 전파된다.

---

# 26. 공식 기준 자료

- [json-render Streaming](https://json-render.dev/docs/streaming)
- [json-render Core API](https://json-render.dev/docs/api/core)
- [json-render Generation Modes](https://json-render.dev/docs/generation-modes)
- [json-render Specs](https://json-render.dev/docs/specs)
- [json-render Catalog](https://json-render.dev/docs/catalog)
- [json-render Validation](https://json-render.dev/docs/validation)
- [RFC 6902: JSON Patch](https://www.rfc-editor.org/rfc/rfc6902.html)
- [NDJSON Specification](https://github.com/ndjson/ndjson-spec)
- [CloudEvents Specification](https://github.com/cloudevents/spec)
