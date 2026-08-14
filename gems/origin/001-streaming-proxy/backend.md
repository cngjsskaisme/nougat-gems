# Streaming Proxy — Backend

## Problem

장시간 생성 작업을 완료한 뒤 한 번에 반환하면 첫 응답까지의 시간이 길어지고, 클라이언트가 연결을 끊어도 서버 작업이 계속될 수 있습니다.

## Architecture

```text
HTTP Controller
    ↓ validate
Application Service
    ↓ async events
Event Sequencer
    ↓
NDJSON Serializer
    ↓
Bounded Stream Writer
    ↓ flush
Reverse Proxy
    ↓
Client
```

취소는 반대 방향으로 전파합니다.

```text
Client Disconnect → HTTP Abort → Service → Upstream Cancellation
```

## Event Contract

한 Event는 한 줄의 JSON으로 직렬화합니다.

```json
{
  "version": "1",
  "requestId": "request-id",
  "sequence": 1,
  "type": "content.text",
  "payload": {},
  "timestamp": "ISO-8601"
}
```

정상 Stream은 Terminal Event를 정확히 한 번 보냅니다. Stream 시작 전 검증 실패는 HTTP 오류로, 시작 후 오류는 안전한 Error Event로 구분합니다.

## Responsibilities

Controller는 HTTP 생명주기만 담당하고 검색·Prompt 조립·Provider 처리 같은 업무 로직을 직접 수행하지 않습니다. Service는 HTTP Response 객체를 모릅니다.

Stream Writer는 쓰기 가능 상태, Backpressure, Flush, Write Failure, 중복 종료를 관리합니다.

## Backpressure

무제한 Buffer를 허용하지 않습니다. 요청별 Queue 상한을 두고 소비가 느리면 upstream 소비를 늦추며, 제어할 수 없는 upstream이면 제한된 Buffer를 넘는 시점에 안전하게 취소합니다.

## Failure / Recovery

- Stream 전 검증 실패 → HTTP 4xx
- Stream 후 업무 오류 → sanitized error event → close
- Client Disconnect → event 전송 시도 없이 upstream 취소
- Terminal Event 이후 추가 Event 금지

## Observability

Payload 원문 대신 requestId, first-event time, event count, byte count, last sequence, duration, abort source, error category 정도만 기록합니다.

## Validation

- 한 Event가 한 줄 NDJSON인지
- Sequence가 단조 증가하는지
- Terminal Event가 정확히 한 번인지
- Disconnect가 upstream 취소로 이어지는지
- 느린 Client에서 Buffer가 상한을 넘지 않는지
