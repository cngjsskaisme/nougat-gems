# Guest HMAC Authentication — Backend

## Intent

짧은 Guest Session을 발급하고 요청마다 Timestamp, Nonce, HMAC Signature를 검증합니다.

## Session Init

공개 Init Endpoint는 무제한 호출되지 않도록 IP/Client 기준 Rate Limit을 둡니다.

```text
Generate guestToken
Generate random session secret
Store {secret} with short TTL
Return token + secret + expiresIn
```

실제 Endpoint 이름, Cache Key Prefix, TTL과 Rate Limit 값은 환경별 Configuration입니다.

## Signed Request

```text
Authorization: Guest <token>
X-Timestamp: <timestamp>
X-Nonce: <nonce>
X-Signature: <signature>
```

Signing Input은 FE/BE가 동일한 Canonicalization Contract를 사용합니다.

```text
METHOD
+ CanonicalPath
+ NormalizedBody
+ Timestamp
+ Nonce
```

## Verification Order

1. Guest Token 형식 확인
2. Session Secret 조회
3. Timestamp Window 확인
4. Nonce를 atomic `SET-if-absent` 형태로 소비
5. Signature 재계산
6. Constant-time compare

Nonce는 짧은 TTL 동안 한 번만 사용 가능해야 합니다.

## Responsibilities

- Controller: HTTP 계약
- Auth Guard/Middleware: 검증 순서와 차단
- Signature Service: canonicalization + HMAC
- Session Repository: TTL/atomic nonce storage
- Rate Limiter: 공개 Init Endpoint 보호

구체 클래스명과 프로젝트 파일 구조는 Gem의 Contract가 아닙니다.

## Validation

- 같은 요청/Nonce 재전송 차단
- Timestamp Window 밖 요청 차단
- Body/Path 변경 시 Signature 실패
- Timing-safe compare 사용
- Session 만료 후 요청 실패
- Init Endpoint Rate Limit
