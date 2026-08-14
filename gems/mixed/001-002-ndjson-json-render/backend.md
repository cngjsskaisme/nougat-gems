# NDJSON × json-render Adapter — Backend

## Contract Gap

Streaming Gem은 순서 있는 Event 전달을 책임지고, UI Gem은 SpecStream과 Catalog Validation을 책임집니다. 둘을 직접 연결하면 Transport Event Type과 Component Type이 섞이거나, 불완전 Spec의 lifecycle이 전송 계약에 드러나지 않을 수 있습니다.

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

Transport의 `type`은 Event 의미이고, UI Component Type은 Patch 내부 `elements[id].type`에만 존재합니다.

## Part Lifecycle

Text와 UI가 교차하는 Inline Output을 Part 단위로 보존합니다.

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

Part ID는 Message 내에서 유일하고, 종료된 Part에는 Event를 추가하지 않습니다.

## UI Part Runtime

각 UI Part는 독립 Compiler를 가집니다.

```text
Patch
  ↓ Patch Guard
Shadow Compiler
  ↓
Incremental Safety Validation
  ↓
content.ui.patch
```

Patch는 RFC 6902 Operation과 허용 Root(`/root`, `/elements`, `/state`)를 검증하고 prototype-pollution 관련 Path를 차단합니다.

## Final Snapshot

UI Part가 끝날 때 전체 Spec을 다시 검증합니다.

```text
compiled spec
  ↓ structural validation
  ↓ catalog validation
  ↓ UI policy validation
validated snapshot
```

검증된 Snapshot만 `content.ui.snapshot`으로 전송합니다. UI Part만 실패하고 Text가 정상이라면 전체 결과를 `partial`로 완료할 수 있습니다.

## Backpressure / Abort

Adapter와 Writer 사이에는 bounded queue를 사용합니다. Client Disconnect는 Mixed Parser, upstream LLM, Compiler, Validation, pending write까지 취소로 전파합니다.

## Validation

- Event Sequence와 Terminal Event
- Text/UI 교차 순서
- Part별 Compiler 격리
- invalid Patch가 Client로 전달되지 않는지
- final snapshot이 전체 validation 후에만 전송되는지
- UI Part failure와 whole-stream failure 구분
