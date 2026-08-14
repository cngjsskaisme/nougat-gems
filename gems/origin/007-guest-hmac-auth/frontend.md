# Guest HMAC Authentication — Frontend

## Intent

Application Code가 Guest Session과 Signature 생성 Detail을 의식하지 않도록 공통 API Transport Layer에서 처리합니다.

## Lifecycle

```text
App Start
  ↓ restore short-lived session if valid
No valid session?
  ↓ init
Store in memory
  ↓
Schedule refresh before expiry
```

새로고침 복원이 꼭 필요하다면 Browser Storage/Cookie 사용 여부를 별도 Threat Model로 결정합니다. Client-accessible Secret을 저장하므로 XSS Risk를 반드시 함께 고려합니다. 실제 Cookie/Storage Key 이름은 공개 Gem에 포함하지 않습니다.

## Request Wrapper

모든 보호 API 요청은 공통 Wrapper를 통과합니다.

```text
ensureSession
→ timestamp
→ nonce
→ canonical request
→ HMAC signature
→ send
```

FE와 BE는 Method, Path, Body Normalize 규칙을 byte-level로 동일하게 정의해야 합니다.

## Retry

401이 Session 만료로 분류되고 아직 응답 본문을 소비하지 않았다면 새 Session을 발급한 뒤 원 요청을 **최대 한 번** 재생성해 시도할 수 있습니다.

서명에 Timestamp와 Nonce가 포함되므로 기존 Signed Request를 그대로 재전송하지 않습니다.

## Boundaries

UI Component는 Token, Secret, Signature를 직접 다루지 않습니다. Session Manager와 API Transport가 책임을 가집니다.

## Validation

- FE/BE Canonicalization Contract Test
- Refresh 동시 호출 Single-flight
- 401 Retry 무한 루프 방지
- 만료 Session 복원 금지
- Secret이 로그/분석 이벤트에 포함되지 않는지 검사
