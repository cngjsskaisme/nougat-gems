# Backend AI Agent Prompt

당신은 Backend Security Architect입니다.

로그인 기능이 없는 서비스에서 사용할 Guest Authentication 시스템을 구현하세요.

이 시스템은 사용자 인증(User Authentication)이 아니라 Guest Client Authentication을 위한 것입니다.

JWT는 사용하지 않습니다.

Redis 기반의 단기 Guest Session과 HMAC Signature를 이용하여 API 요청을 보호합니다.

---

## 목표

다음 공격을 방어합니다.

* API 남용
* Replay Attack
* Request Tampering
* Authorization Header 위조
* Secret 장기 탈취

---

## Guest Session 발급 API

공개 API

POST /api/guest/init

인증 없이 접근 가능합니다.

호출 시

* guestToken(UUID)
* secret(32byte random)

을 생성합니다.

응답

```json
{
  "guestToken": "...",
  "secret": "...",
  "expiresIn": 300
}
```

Redis 저장

Key

```
guest:{guestToken}
```

Value

```json
{
  "secret": "..."
}
```

TTL

```
300초
```

---

## Rate Limit

/api/guest/init

IP 기준

분당 5회

초과 시

```
429 Too Many Requests
```

---

## 인증 대상

/api/guest/init

제외

모든 API는 Guest 인증이 필요합니다.

Authorization Header

```
Authorization: Guest {guestToken}
```

---

## Request Header

모든 요청은

* Authorization
* X-Timestamp
* X-Nonce
* X-Signature

를 포함해야 합니다.

---

## Signature

Signature 생성 알고리즘

```
HMAC-SHA256(secret)
```

Signing String

```
METHOD
+
CanonicalPath
+
NormalizedBody
+
Timestamp
+
Nonce
```

---

## 인증 절차

Guard 또는 Middleware에서

순서대로 수행합니다.

1.

Authorization 확인

2.

Redis

```
guest:{token}
```

조회

없으면

401

3.

Timestamp

허용 범위

±30초

4.

Nonce

Redis

```
nonce:{token}:{nonce}
```

SET NX

TTL 30초

이미 존재하면

401

5.

Signature 재계산

6.

timingSafeEqual()

비교

다르면

401

---

## Replay Attack

Nonce는

한 번만 사용 가능합니다.

Timestamp는

±30초

이내만 허용합니다.

---

## Security

반드시 사용

* randomUUID()
* randomBytes(32)
* crypto.createHmac()
* crypto.timingSafeEqual()
* Redis TTL
* Redis SET NX

---

## 아키텍처

다음 구조를 권장합니다.

* GuestAuthController
* GuestAuthService
* SecurityAuthGuard
* RateLimitMiddleware
* SignatureService
* RedisRepository

Controller는 요청만 처리합니다.

인증 로직은 Guard

Signature 계산은 Service

Redis 접근은 Repository

로 분리합니다.

---

## 구현 품질

* SOLID
* TypeScript Strict
* Unit Test 가능
* DI 기반 구조
* Signature 로직 재사용 가능
* 에러 응답 일관성 유지
* Canonical Path/Body 생성 함수 분리
