# NDJSON × json-render Adapter — Frontend

## Core Boundary

NDJSON Parser와 UI Renderer를 직접 연결하지 않습니다.

```text
NDJSON Event
  ↓ Contract + Sequence Guard
Part Store
  ↓
UI Patch Compiler
  ↓
workingSpec
  ↓ Boundary Validation
renderSpec (Last-Known-Good)
  ↓
Renderer
```

## Working Spec vs Render Spec

`workingSpec`은 정상 Patch를 모두 적용한 최신 상태입니다. 하지만 Streaming 중에는 Root나 Child 참조가 아직 완성되지 않아 렌더링 불가능할 수 있습니다.

`renderSpec`은 Frontend Boundary Validation을 통과한 마지막 안전 상태입니다. 최신 Patch가 불완전하다면 기존 `renderSpec`을 유지합니다.

## Snapshot Reconciliation

Backend의 최종 Snapshot은 해당 UI Part의 최종 권위 상태로 취급하되 Frontend에서도 다시 검증합니다.

```text
snapshot
  ↓ version / catalog / boundary validation
valid   → renderSpec = snapshot
invalid → keep LKG + mark part failed
```

Snapshot은 동일 연결에서 상태를 동기화하는 수단이지, 끊긴 Stream을 자동 Resume하는 프로토콜은 아닙니다.

## Sequence Guard

요청별 Sequence를 엄격하게 검사합니다. 중복·역순·누락은 기본적으로 Protocol Error입니다. 이미 일부 UI를 반영한 뒤 전체 요청을 자동 재시도하지 않습니다.

## Part Failure Isolation

UI Part 하나가 실패해도 완료된 Text Part와 다른 안전한 Part는 유지할 수 있습니다. Fallback은 해당 Part 범위에만 표시합니다.

## Security

- Event/Patch Runtime Schema 검증
- prototype-pollution Path 차단
- Registry 밖 Component 금지
- Catalog 밖 Action 금지
- arbitrary HTML/script 금지
- UI Element/State 크기 제한
- server-authorized action의 서버 재검증

## Validation

- 첫 Patch는 불완전, 이후 안전한 Spec으로 Promote
- 유효한 UI 뒤 invalid Patch → LKG 유지
- Patch 결과와 Snapshot 불일치 → valid Snapshot으로 reconcile
- Snapshot 누락/Version mismatch
- Terminal Event 없는 EOF
- Abort와 Network Failure 구분
