# 스트리밍 프록시 FE 프로그램 설계서

## 1. 문서 목적

본 문서는 Nuxt 기반 프론트엔드 애플리케이션에서 백엔드의 NDJSON 스트림을 브라우저까지 손실 없이 전달하고, 브라우저가 이를 점진적으로 파싱하여 UI 이벤트로 변환하기 위한 설계를 정의한다.

프론트엔드 범위에는 다음 두 실행 영역이 포함된다.

```text
Browser Runtime
Nuxt/Nitro Server Runtime
```

Nitro는 프론트엔드 애플리케이션이 소유하는 서버 런타임으로서 Reverse Proxy 역할을 수행한다.

---

## 2. 설계 범위

### 포함 범위

- 브라우저 Streaming API Client
- `ReadableStream` 기반 Chunk 읽기
- UTF-8 증분 디코딩
- NDJSON Buffer와 줄 단위 파싱
- Stream Event Dispatcher
- `AbortController` 기반 취소
- Nitro `routeRules` Reverse Proxy
- 브라우저/SSR 실행 환경별 Endpoint 선택
- 인증 헤더 및 401 재시도 연계
- 프록시 및 중간 계층의 버퍼링 방지

### 제외 범위

- 채팅 화면 상세 구성
- 업무 이벤트별 UI 컴포넌트 설계
- 백엔드 LLM 호출 및 이벤트 생성
- 암복호화
- 대화 저장 및 RAG 처리

---

## 3. 목표 구조

```text
UI / Chat State
    ↓
Streaming API Facade
    ↓
Client Transport
    ├─ Request Builder
    ├─ Authentication Adapter
    ├─ Abort Manager
    └─ Retry Coordinator
    ↓
fetch()
    ↓
/api/**
    ↓
Nitro routeRules Reverse Proxy
    ↓
Backend Streaming Endpoint
    ↓
NDJSON Chunks
    ↓
ReadableStream Reader
    ↓
Incremental Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓
Event Dispatcher
    ↓
Chat State / Rendering
```

---

## 4. 주요 컴포넌트

### 4.1 Streaming API Facade

상위 채팅 계층에 스트리밍 호출 인터페이스를 제공한다.

**책임**

- 요청 모델 수신
- 스트림 시작 및 종료 상태 전달
- 이벤트 콜백 또는 Async Iterator 제공
- 취소 핸들 반환
- Transport 오류를 애플리케이션 오류로 변환

상위 계층은 `fetch`, `ReadableStream`, Buffer 처리 방식을 직접 알지 못하도록 한다.

### 4.2 Client Transport

브라우저에서 실제 HTTP 요청을 수행한다.

**책임**

- Browser Runtime에서는 상대 경로 `/api` 사용
- SSR Runtime에서는 내부 Backend Base URL 또는 런타임 설정 사용
- 요청 Header 구성
- 응답 상태 및 Content-Type 검증
- `Response.body` 노출
- 취소 신호 연결

스트리밍 요청에는 전체 응답을 메모리에 적재하는 고수준 응답 변환 함수를 사용하지 않는다.

### 4.3 Authentication Adapter

전송에 필요한 인증 메타데이터를 주입한다.

예시 논리 항목은 다음과 같다.

```text
Authorization
Timestamp
Nonce
Signature
```

SSR 내부 요청은 별도의 내부 요청 식별 Header를 사용할 수 있다.

인증 생성 로직은 Streaming Parser와 분리한다.

### 4.4 Retry Coordinator

인증 만료 등 재시도 가능한 오류를 처리한다.

```text
Initial Request
    ↓ 401
Guest/User Session Initialize or Refresh
    ↓
Create New Request
    ↓
Retry Once
```

이미 일부 스트림 이벤트를 UI에 반영한 뒤에는 요청 전체를 자동 재시도하지 않는다. 자동 재시도는 응답 본문 소비 전의 인증 실패에 한정한다.

### 4.5 Stream Reader

`Response.body`에서 Chunk를 순차적으로 읽는다.

```text
Response.body
    ↓
getReader()
    ↓
read()
    ↓
Uint8Array Chunk
```

빈 본문, 비스트리밍 응답 및 조기 종료를 명확히 구분한다.

### 4.6 Incremental Decoder

Chunk 경계와 UTF-8 문자 경계가 일치하지 않으므로 증분 디코딩을 사용한다.

```text
Uint8Array Chunk
    ↓
TextDecoder(stream mode)
    ↓
Decoded Text Fragment
```

스트림 종료 시 Decoder의 남은 내부 버퍼를 Flush한다.

### 4.7 Line Buffer

NDJSON 한 줄이 여러 Chunk로 분리되거나 하나의 Chunk에 여러 줄이 포함되는 상황을 처리한다.

```text
buffer += decodedFragment
lines = split by newline
completeLines = all except last
buffer = last incomplete line
```

완성된 줄만 Parser에 전달하고 마지막 미완성 줄은 다음 Chunk까지 보존한다.

### 4.8 NDJSON Parser

완성된 각 줄을 독립 JSON Event로 파싱한다.

**책임**

- 빈 줄 무시
- JSON 파싱
- Event Contract 검증
- 알 수 없는 이벤트 유형 처리
- 파싱 오류 위치와 요청 식별자 기록

한 줄의 오류가 전체 연결 종료로 이어질지, 오류 이벤트로 전환할지는 계약 정책으로 결정한다. 기본값은 프로토콜 오류로 스트림을 종료하는 것이다.

### 4.9 Event Dispatcher

파싱된 이벤트를 채팅 상태 계층에 전달한다.

대표적인 논리 이벤트는 다음과 같다.

```text
meta
text
block
dealCards
comparison
questions
done
error
```

Event Dispatcher는 렌더링 컴포넌트를 직접 호출하지 않고 상태 변경 명령으로 변환한다.

### 4.10 Abort Manager

사용자 중지, 화면 이탈, 새 요청 시작 등의 상황에서 스트림을 취소한다.

```text
UI Stop Action
    ↓
AbortController.abort()
    ↓
fetch cancellation
    ↓
Nitro connection close
    ↓
Backend cancellation propagation
```

취소는 일반 오류와 구분하여 UI에 불필요한 오류 메시지를 표시하지 않는다.

### 4.11 Nitro Reverse Proxy

Nuxt 서버 런타임에서 `/api/**` 요청을 백엔드로 전달한다.

```text
Browser /api/deals/chat
    ↓
Nitro routeRules
    ↓
Backend /api/deals/chat
```

**책임**

- 동일 출처 API 제공
- CORS 경계 단순화
- 요청 Header와 Body 전달
- 응답 Status와 Header 전달
- Streaming Body를 재조립하지 않고 전달
- 캐시 및 압축 정책 조정

별도의 Nuxt API Handler가 응답 전체를 읽고 다시 반환하는 구조는 피한다.

---

## 5. 전체 요청 시퀀스

```text
User
    ↓
Chat State
    ↓
Streaming API Facade
    ↓
Client Transport
    ↓ fetch /api/**
Nitro routeRules Reverse Proxy
    ↓
Backend Streaming Endpoint
    ↓ NDJSON chunks
Nitro Reverse Proxy
    ↓
ReadableStream Reader
    ↓
Incremental Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓
Event Dispatcher
    ↓
Chat State / UI
```

---

## 6. 스트림 상태 모델

```text
idle
  ↓ start
connecting
  ↓ headers received
streaming
  ↓ done event or EOF
completed
```

예외 상태는 다음과 같다.

```text
connecting → failed
streaming → failed
connecting → cancelled
streaming → cancelled
```

### 상태별 원칙

- `connecting`: 응답 Header가 오기 전
- `streaming`: 하나 이상의 Chunk를 처리 중
- `completed`: 정상 `done` 이벤트 또는 합의된 정상 EOF
- `failed`: 네트워크, HTTP, 프로토콜 또는 파싱 오류
- `cancelled`: 사용자가 의도적으로 중단

---

## 7. NDJSON 계약 처리

한 Event는 반드시 한 줄의 JSON으로 표현한다.

```text
{event 1}\n
{event 2}\n
{event 3}\n
```

### FE 검증 항목

```text
type
requestId or conversationId(optional)
sequence(optional)
payload
timestamp(optional)
```

`sequence`가 제공되는 경우 중복, 역순 및 누락 이벤트를 탐지할 수 있다.

스트림 종료 시 Buffer에 남은 텍스트가 있다면 마지막 완성 Event인지 검사한다. 불완전한 JSON이면 프로토콜 오류로 처리한다.

---

## 8. 오류 및 재시도 정책

| 구간 | 오류 | 처리 원칙 |
|---|---|---|
| 요청 전 | 인증 정보 생성 실패 | 전송 중단 |
| 응답 전 | 401 | 세션 초기화/갱신 후 1회 재시도 |
| 응답 전 | 4xx | 업무 오류로 변환, 자동 재시도 금지 |
| 응답 전 | 5xx | 재시도 가능 상태로 표시하되 자동 반복 금지 |
| Streaming | 연결 중단 | 수신 완료 내용 보존 여부를 이벤트 계약에 따라 결정 |
| Streaming | 잘못된 JSON | 스트림 종료 및 프로토콜 오류 |
| Streaming | `error` 이벤트 | 서버 오류 메시지를 안전한 UI 메시지로 변환 |
| Streaming | `done` 없이 EOF | 비정상 종료로 처리하는 것을 기본값으로 함 |
| User Action | Abort | `cancelled` 상태로 종료 |

부분 응답이 이미 표시된 경우 동일 요청을 자동 재시도하면 중복 메시지가 생길 수 있으므로 사용자의 명시적 재요청을 원칙으로 한다.

---

## 9. 프록시 설계 요구사항

### Streaming 보존

- Proxy가 응답 전체를 Buffering하지 않아야 한다.
- 응답 Body를 JSON 객체로 변환하지 않아야 한다.
- Content-Length 강제 계산을 피한다.
- NDJSON Content-Type을 그대로 전달한다.
- 중간 웹서버의 Proxy Buffering을 비활성화한다.

### Header 정책

전달 대상 예시는 다음과 같다.

```text
Content-Type
Cache-Control
Connection-related streaming headers where applicable
Request/Trace ID
```

Hop-by-Hop Header는 런타임 및 Proxy 표준에 따라 정리한다.

### 캐시 정책

Streaming API는 캐시하지 않는다.

```text
Cache-Control: no-store
```

### 환경 설정

Proxy Target은 배포 환경별로 달라질 수 있다.

```text
Local Backend
Development Backend
Staging Backend
Production Backend
```

`routeRules`가 빌드 시점 설정을 사용하는 경우, 필요한 환경변수가 빌드 전에 제공되어야 한다.

---

## 10. 운영 배치 구조

```text
Browser
    ↓ HTTPS
Apache / Nginx / Load Balancer
    ↓
Nuxt/Nitro
    ↓ routeRules proxy
Backend
```

각 중간 계층에서 다음을 확인한다.

- 응답 Buffering 비활성화
- Idle Timeout이 최대 스트림 시간보다 길 것
- Chunk Flush 지연이 없을 것
- 압축이 작은 Chunk 전달을 과도하게 지연시키지 않을 것
- 연결 종료가 Backend 취소로 전파될 것

---

## 11. 관측성

FE에서 기록 가능한 메타데이터는 다음과 같다.

```text
requestId
connection start time
first-byte time
first-event time
event count
last sequence
stream duration
completion status
error category
abort reason
```

실제 사용자 메시지와 Event Payload는 운영 로그에 기록하지 않는 것을 기본 원칙으로 한다.

---

## 12. 비기능 요구사항

### 응답성

- 첫 Event가 도착하는 즉시 상태에 반영한다.
- 전체 응답 완료를 기다리지 않는다.
- Chunk마다 과도한 전체 메시지 재렌더링을 피한다.

### 안정성

- 여러 동시 스트림의 Buffer와 AbortController를 분리한다.
- 화면 종료 시 열린 Reader를 정리한다.
- `done`, EOF, Abort를 중복 종료하지 않도록 한다.

### 호환성

- 지원 브라우저에서 `fetch`, `ReadableStream`, `TextDecoder`, `AbortController` 사용 가능 여부를 확인한다.
- SSR에서는 Browser 전용 API를 호출하지 않는다.

---

## 13. 테스트 설계

### 단위 테스트

- 한 Event가 여러 Chunk로 분할된 경우
- 하나의 Chunk에 여러 Event가 포함된 경우
- 멀티바이트 문자가 Chunk 경계에서 분리된 경우
- 마지막 줄 개행 유무
- 잘못된 JSON과 알 수 없는 Event Type
- Abort 상태 전이

### 통합 테스트

- Browser → Nitro → Backend 전체 스트리밍
- 401 이후 1회 재시도
- Proxy 환경별 Target 전환
- 중간 연결 종료의 Backend 전파
- 장시간 스트림 Timeout

### 운영 검증

- Nginx/Apache/Load Balancer Buffering 여부
- 첫 Event 지연 시간
- 배포 환경별 Content-Type과 Cache-Control
- 동시 연결 수 증가 시 메모리와 연결 안정성

---

## 14. 완료 기준

- 브라우저가 `Response.body`를 직접 읽어 Event를 점진적으로 처리한다.
- Chunk 경계와 무관하게 NDJSON Event가 정확히 복원된다.
- Nitro Proxy가 응답을 Buffering하거나 재조립하지 않는다.
- 사용자 취소가 Browser에서 Backend까지 전파된다.
- 401 재시도는 스트림 소비 전에만 1회 수행된다.
- 정상 완료, 오류, EOF, 취소 상태가 명확히 구분된다.
- 배포 중간 계층에서도 Streaming 지연이 발생하지 않는다.
