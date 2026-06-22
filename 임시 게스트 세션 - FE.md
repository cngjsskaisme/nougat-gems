# Frontend AI Agent Prompt

당신은 Frontend Security Engineer입니다.

Backend의 Guest Authentication 시스템을 사용하는 Frontend 인증 레이어를 구현하세요.

목표는 사용자가 Guest Session을 의식하지 않고 API를 사용할 수 있도록 만드는 것입니다.

---

## Guest Session

Backend

```
POST /api/guest/init
```

응답

```json
{
  "guestToken": "...",
  "secret": "...",
  "expiresIn": 300
}
```

---

## 앱 시작

앱 시작 시

```
ensureGuestSession()
```

을 자동 실행합니다.

순서

1.

Cookie 확인

Guest Session 존재

만료되지 않았으면 복원

2.

없으면

```
POST /guest/init
```

3.

응답 저장

* Memory
* Cookie

4.

자동 갱신 예약

---

## Cookie

예시 이름

```
nougat_guest_session_v1
```

저장 내용

```json
{
  "token": "...",
  "secret": "...",
  "expiresAt": 123456789
}
```

옵션

* SameSite=Lax
* Secure(HTTPS)

---

## 자동 갱신

Guest Session 만료

30초 전

자동으로

```
POST /guest/init
```

호출

새 Session으로 교체합니다.

---

## API Wrapper

모든 API 요청은

공통 Wrapper를 사용합니다.

예시

```
apiFetch()
```

자동 수행

* Guest Session 확인
* Authorization 생성
* Timestamp 생성
* Nonce 생성
* Signature 생성

Header

```
Authorization: Guest {token}

X-Timestamp

X-Nonce

X-Signature
```

---

## Signature

Backend와 동일한 규칙

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

## 401 처리

401 발생 시

Guest Session 재발급

```
POST /guest/init
```

수행

성공하면

원래 요청

1회만 재시도합니다.

무한 재시도는 금지합니다.

---

## Session 관리

Memory를 우선 사용합니다.

페이지 새로고침 시

Cookie에서 복원합니다.

Memory와 Cookie는 항상 동기화합니다.

---

## 구현 구조

권장 구조

```
plugins/

guest-init.client.ts

composables/

useGuestSession.ts

useApi.ts

utils/

signature.ts

cookie.ts
```

---

## 구현 품질

* TypeScript Strict
* Composition API
* API Wrapper 재사용
* Signature Utility 분리
* Session 관리 모듈화
* 테스트 가능한 구조
* Backend와 Signature 규칙 100% 동일
