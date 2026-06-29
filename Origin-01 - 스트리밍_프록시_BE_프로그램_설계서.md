# 스트리밍 프록시 BE 프로그램 설계서

## 1. 문서 목적

본 문서는 채팅 백엔드가 LLM 및 내부 처리 결과를 NDJSON Event로 점진적으로 생성하고, 프론트엔드 Reverse Proxy를 통해 브라우저까지 안정적으로 전달하기 위한 서버 측 설계를 정의한다.

백엔드는 전체 응답 생성을 기다리지 않고 Event가 준비되는 즉시 Flush하며, 클라이언트 연결 종료를 내부 작업 취소로 전파한다.

---

## 2. 설계 범위

### 포함 범위

- Streaming HTTP Controller
- NDJSON 응답 Header
- Event Contract
- LLM Chunk 수신 및 변환
- Semantic/Event Parser
- NDJSON 직렬화와 Flush
- Backpressure 처리
- Client Disconnect 및 Abort 전파
- 정상 완료와 오류 Event
- Proxy 친화적 응답 정책
- 관측성 및 운영 Timeout

### 제외 범위

- 브라우저 `ReadableStream` 파싱
- Nitro `routeRules` 설정 상세
- 채팅 UI 렌더링
- 암복호화
- 검색 및 RAG 세부 로직
- LLM Prompt 내용

---

## 3. 목표 구조

```text
HTTP Streaming Controller
    ↓
Request Validator
    ↓
Chat Service
    ↓
Chat Orchestrator / LLM Gateway
    ↓ token or semantic chunk
Section/Event Parser
    ↓ structured event
Event Sequencer
    ↓
NDJSON Serializer
    ↓
Stream Writer / Flush
    ↓
Nitro Reverse Proxy
    ↓
Browser
```

취소 경로는 반대 방향으로 전파한다.

```text
Browser Disconnect
    ↓
Nitro Upstream Connection Close
    ↓
HTTP Controller Abort Signal
    ↓
Chat Service
    ↓
LLM Gateway Cancellation
```

---

## 4. 주요 컴포넌트

### 4.1 Streaming HTTP Controller

Streaming Endpoint의 HTTP 생명주기를 담당한다.

**책임**

- 요청 검증
- Streaming Response Header 설정
- Abort Signal 생성 및 연결
- Chat Service의 Event Stream 구독
- NDJSON Write와 Flush
- 연결 종료 및 리소스 정리

Controller는 검색, Prompt 조립, LLM 공급자별 처리와 같은 업무 로직을 직접 수행하지 않는다.

### 4.2 Request Validator

Streaming 시작 전에 요청의 필수 값을 검증한다.

```text
conversationId(optional)
message
mode(optional)
client capabilities(optional)
requestId
```

응답 Header를 전송한 뒤에는 일반 JSON 오류 응답으로 전환하기 어려우므로, 가능한 검증은 스트림 시작 전에 완료한다.

### 4.3 Chat Service Stream Interface

Controller에 구조화된 Event Stream을 제공한다.

권장 추상 형태는 다음과 같다.

```text
streamChat(request, abortSignal) → Async Event Stream
```

Chat Service는 HTTP Response 객체를 직접 다루지 않는다. 이를 통해 Streaming 로직과 업무 로직을 분리하고 테스트 가능성을 확보한다.

### 4.4 LLM Gateway

공급자의 Streaming API를 공통 Chunk Stream으로 추상화한다.

```text
Provider Token/Chunk
    ↓
Normalized LLM Chunk
```

Abort Signal을 공급자 SDK 또는 HTTP 요청 취소 기능에 연결한다.

### 4.5 Section/Event Parser

LLM의 Raw Text 또는 Chunk를 프론트엔드가 이해할 수 있는 의미 Event로 변환한다.

```text
Raw Token Stream
    ↓
Incremental Section Parser
    ↓
Semantic Block
    ↓
Event
```

대표 Event는 다음과 같다.

```text
meta
text
block/dealCards
block/comparison
block/questions
done
error
```

LLM Raw Text를 그대로 UI 계약으로 사용하지 않고 Parser가 안정된 Event Contract를 생성한다.

### 4.6 Event Sequencer

각 Event에 요청 단위 순번을 부여한다.

```text
requestId
sequence
 type
payload
```

순번은 FE가 중복, 역순 및 누락을 감지하는 데 사용할 수 있다.

### 4.7 NDJSON Serializer

각 Event를 단일 JSON 한 줄로 직렬화한다.

```text
serialize(event) + "\n"
```

Event Payload 내부에 줄바꿈이 있더라도 JSON 문자열 Escape를 사용하므로 한 Event는 물리적으로 한 줄을 유지한다.

### 4.8 Stream Writer

직렬화된 Event를 응답 스트림에 기록하고 가능한 즉시 Flush한다.

**책임**

- 쓰기 가능 상태 확인
- Backpressure 대기
- Write 실패 감지
- Flush
- 종료 중복 방지
- 연결 종료 시 Abort 발생

### 4.9 Abort Coordinator

다음 취소 원인을 하나의 Abort Signal로 통합한다.

```text
client disconnect
user cancellation
gateway timeout
server shutdown
internal policy cancellation
```

Abort 발생 시 검색, DB 조회, LLM 호출, Parser 및 후처리를 가능한 범위에서 모두 중단한다.

---

## 5. HTTP 응답 설계

### 권장 응답 Header

```text
Content-Type: application/x-ndjson; charset=utf-8
Cache-Control: no-store, no-transform
X-Content-Type-Options: nosniff
```

실제 프레임워크와 인프라에 따라 Streaming 및 Buffering 방지 Header를 추가할 수 있다.

### 응답 상태

- 요청 검증 실패는 스트림 시작 전 4xx로 반환한다.
- 스트림이 시작된 뒤 발생한 오류는 `error` Event로 전달한다.
- 치명적 연결 오류 시 더 이상 Event를 쓸 수 없으므로 연결을 종료한다.

### Content-Length

전체 크기를 사전에 알 수 없으므로 설정하지 않는다.

---

## 6. Event Contract

모든 Event는 공통 Envelope를 사용한다.

```json
{
  "version": "1",
  "requestId": "request-id",
  "sequence": 1,
  "type": "text",
  "payload": {},
  "timestamp": "ISO-8601"
}
```

### Event 유형

| 유형 | 목적 |
|---|---|
| `meta` | 대화 ID, 모델, 처리 모드 등 초기 메타데이터 |
| `text` | 점진적 텍스트 조각 |
| `block/*` | 구조화된 UI 블록 |
| `done` | 정상 완료와 최종 메타데이터 |
| `error` | 스트림 시작 후 발생한 안전한 오류 정보 |

### 완료 규칙

정상 스트림은 정확히 한 개의 `done` Event를 마지막에 전송한다.

```text
0..N data events
    ↓
1 done event
    ↓
stream close
```

오류 Event를 전송한 경우 후속 정책은 다음 중 하나로 고정한다.

```text
error → stream close
```

오류 이후 정상 Event를 계속 보내지 않는 것을 기본 원칙으로 한다.

---

## 7. 정상 처리 시퀀스

```text
Browser
    ↓ POST streaming request
Nitro Proxy
    ↓
Streaming Controller
    ↓ validate request
Chat Service
    ↓
LLM Gateway
    ↓ chunk
Event Parser
    ↓ event
Event Sequencer
    ↓
NDJSON Serializer
    ↓ line
Stream Writer
    ↓ flush
Nitro Proxy
    ↓
Browser
    ⋮
Stream Writer
    ↓ done event
Browser
```

첫 Event는 대화 식별자와 스트림 준비 상태를 담은 `meta` Event로 구성할 수 있다.

---

## 8. 오류 처리 시퀀스

### 스트림 시작 전 오류

```text
Request
    ↓
Validation Failure
    ↓
HTTP 4xx JSON Error
```

### 스트림 시작 후 업무 오류

```text
Streaming Started
    ↓
Service / LLM Error
    ↓
Sanitized error Event
    ↓
Flush
    ↓
Stream Close
```

### 연결 종료 오류

```text
Write Failure or Client Disconnect
    ↓
Abort Signal
    ↓
Cancel LLM and Internal Tasks
    ↓
Release Resources
```

연결이 이미 종료된 경우 오류 Event를 추가로 쓰지 않는다.

---

## 9. Backpressure 설계

클라이언트 또는 Proxy가 데이터를 소비하는 속도가 서버 생성 속도보다 느릴 수 있다.

**처리 원칙**

- Stream Writer의 쓰기 결과를 확인한다.
- 내부 Buffer가 임계치를 넘으면 다음 Event 생성을 일시 정지한다.
- LLM 공급자가 일시 정지를 지원하지 않는 경우 제한된 Queue를 사용한다.
- Queue 한도를 초과하면 스트림을 안전하게 종료하고 관련 작업을 취소한다.
- 무제한 메모리 Buffer를 허용하지 않는다.

Backpressure는 요청별 메모리 사용량과 서버 전체 동시 연결 수를 함께 고려한다.

---

## 10. Abort 및 리소스 정리

### 취소 전파 대상

```text
LLM streaming request
search request
DB query where supported
RAG retrieval
summary generation
parser loop
pending stream write
```

### 정리 항목

- Pending Message 상태 정리
- 열린 Reader/Writer 해제
- Timer 제거
- 임시 Buffer 제거
- 요청 단위 Context 제거
- 중복 Finalize 방지

클라이언트 취소를 정상 완료로 저장하지 않는다. 다만 부분 응답 저장 여부는 별도의 대화 저장 정책으로 결정한다.

---

## 11. Proxy 호환 설계

백엔드는 중간 Proxy가 스트림을 그대로 전달할 수 있도록 다음을 준수한다.

- NDJSON Content-Type 사용
- Event마다 개행 포함
- Event 생성 직후 Flush
- 전체 응답을 객체로 모아 한 번에 반환하지 않음
- Cache 금지
- 변환 및 압축으로 인한 지연 최소화
- 장시간 무출력 구간을 피하거나 Heartbeat 정책 적용

### Heartbeat

LLM 또는 검색 처리 중 장시간 Event가 없을 수 있다면 선택적으로 Heartbeat Event를 사용할 수 있다.

```text
heartbeat
```

Heartbeat는 UI 데이터와 분리하며, Proxy Idle Timeout 방지 목적에만 사용한다. 주기는 인프라 Timeout보다 짧게 설정한다.

---

## 12. Timeout 정책

구분 가능한 Timeout은 다음과 같다.

```text
request validation timeout
first-event timeout
idle timeout
overall stream timeout
upstream LLM timeout
write timeout
```

Timeout 발생 시 Abort Coordinator를 통해 관련 작업을 모두 중단한다.

`overall stream timeout`은 사용자 경험과 LLM 최대 처리 시간을 고려해 설정하고, Proxy와 Load Balancer의 Timeout보다 짧게 유지한다.

---

## 13. 관측성

요청 단위로 다음을 측정한다.

```text
requestId
conversationId(optional)
stream start time
first event time
first text time
event count
byte count
last sequence
LLM duration
stream duration
completion status
abort source
error category
```

Event Payload 원문과 사용자 프롬프트는 기본 로그에서 제외한다.

### 핵심 지표

- Time to First Event
- Time to First Text
- 정상 완료율
- Client Abort 비율
- Proxy/Network Disconnect 비율
- LLM 오류율
- Event 파싱 오류율
- 요청당 최대 Buffer 크기
- 동시 Streaming 연결 수

---

## 14. 비기능 요구사항

### 확장성

- Controller는 무상태로 유지한다.
- 요청별 상태는 연결 수명 동안만 유지한다.
- 동시 연결 수에 맞춰 File Descriptor, Thread/Event Loop 및 메모리 한도를 조정한다.

### 안정성

- `done`, `error`, close가 중복 호출되지 않도록 종료 상태를 단일 관리한다.
- 하나의 요청 오류가 다른 요청 스트림에 영향을 주지 않도록 격리한다.
- 서버 종료 시 신규 연결을 차단하고 기존 스트림을 제한 시간 내 정리한다.

### 일관성

- Event Contract는 버전 관리한다.
- Event 순서를 보장한다.
- 구조화 Block Event는 Schema 검증 후 전송한다.

---

## 15. 테스트 설계

### 단위 테스트

- Event 직렬화가 한 줄 NDJSON을 유지하는지 확인
- Event Sequence 증가
- 정상 `done` 1회 전송
- 오류 후 추가 Event 미전송
- Abort 시 내부 작업 취소
- Backpressure Queue 한도

### 통합 테스트

- LLM Chunk → Parser → NDJSON → Client 전체 흐름
- Client Disconnect의 LLM 취소 전파
- Nitro Proxy를 통한 Chunk 즉시 전달
- 긴 무출력 구간과 Heartbeat
- Upstream LLM Timeout
- 다중 동시 스트림

### 장애 테스트

- Proxy 연결 강제 종료
- 느린 클라이언트
- LLM 중간 오류
- JSON 직렬화 실패
- 서버 종료 중 열린 스트림
- Queue Overflow

### 운영 검증

- Proxy Buffering 비활성화
- Load Balancer Idle Timeout
- 첫 Event 및 Chunk Flush 지연
- 동시 연결 증가에 따른 메모리 사용량

---

## 16. 완료 기준

- 요청 검증 후 NDJSON Streaming Header가 설정된다.
- LLM 또는 내부 Event가 생성되는 즉시 한 줄 단위로 Flush된다.
- Event Contract와 Sequence가 모든 응답에서 일관된다.
- 정상 스트림은 정확히 한 개의 `done` Event로 종료된다.
- 스트림 시작 후 오류는 안전한 `error` Event 후 종료된다.
- Client Disconnect가 LLM 및 내부 작업 취소로 전파된다.
- 느린 클라이언트에 대해 무제한 Buffer가 생성되지 않는다.
- Nitro 및 외부 Proxy를 거쳐도 실시간 Chunk 전달이 유지된다.
