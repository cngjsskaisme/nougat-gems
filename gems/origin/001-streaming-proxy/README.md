# 💎 Origin 001 — Streaming Proxy

LLM 또는 장시간 작업 결과를 **NDJSON Event Stream**으로 점진적으로 전달하면서 Abort, Backpressure, Retry 경계를 분리하는 Gem입니다.

## Intent

- 전체 응답 완료를 기다리지 않고 Event가 준비되는 즉시 전달
- Transport와 Business Logic 분리
- Client Disconnect를 내부 작업 취소로 전파
- Chunk 경계와 Event 경계를 분리
- 부분 응답 이후 무분별한 자동 재시도 방지

## Files

- [Backend](./backend.md)
- [Frontend](./frontend.md)
- [Metadata](./metadata.yaml)
