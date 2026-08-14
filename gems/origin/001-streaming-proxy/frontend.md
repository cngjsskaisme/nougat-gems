# Streaming Proxy — Frontend

## Architecture

```text
Application State
    ↓
Streaming API Facade
    ↓
Transport
    ↓ fetch
ReadableStream
    ↓
Incremental UTF-8 Decoder
    ↓
Line Buffer
    ↓
NDJSON Parser
    ↓
Event Dispatcher
    ↓
Application State / Rendering
```

## Chunk ≠ Event

네트워크 Chunk와 UTF-8 문자, NDJSON Line의 경계는 일치하지 않습니다. 따라서 `TextDecoder`를 증분 모드로 사용하고 마지막 미완성 줄을 다음 Chunk까지 보존합니다.

```text
buffer += decodedFragment
completeLines = all lines except last
buffer = last incomplete line
```

완성된 줄만 JSON으로 파싱합니다.

## Proxy Boundary

브라우저는 동일 출처의 일반화된 API 경로를 호출하고, Server Runtime의 Reverse Proxy가 Backend로 전달할 수 있습니다. Proxy는 Response Body를 객체로 재조립하거나 전체 Buffering하지 않아야 합니다.

실제 내부 Host, Backend URL, 배포 경로는 Gem의 Contract가 아니며 프로젝트 Runtime Config에서 결정합니다.

## Retry

응답 Body를 소비하기 전의 인증 실패처럼 안전한 경우에만 제한적으로 재시도합니다. 이미 일부 Event를 UI에 반영했다면 자동 전체 재시도는 중복 상태를 만들 수 있으므로 금지하는 것이 기본입니다.

## Abort

사용자 중지, 화면 이탈, 새 요청 시작 시 `AbortController`를 통해 Reader와 fetch를 취소하고 Backend까지 연결 종료가 전파되도록 합니다.

## Validation

- 한 Event가 여러 Chunk로 나뉜 경우
- 한 Chunk에 여러 Event가 들어온 경우
- 멀티바이트 문자가 Chunk 경계에서 나뉜 경우
- 마지막 개행이 없는 경우
- invalid JSON / unknown event
- Abort와 비정상 EOF 구분
