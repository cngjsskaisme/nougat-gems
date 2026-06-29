# 암복호화 FE 프로그램 설계서

## 1. 문서 목적

본 문서는 채팅 프론트엔드에서 사용자의 프롬프트와 서버 응답을 보호하기 위한 **애플리케이션 계층 하이브리드 암호화 구조**를 정의한다.

암호화 보호 범위는 다음과 같다.

```text
Browser ↔ Chat Backend
```

브라우저에서 생성한 대칭키로 실제 데이터를 암호화하고, 해당 대칭키는 서버의 공개키로 암호화하여 전달한다. 이 구조는 전송 계층의 HTTPS를 대체하지 않으며, HTTPS와 함께 적용하는 것을 전제로 한다.

---

## 2. 설계 범위

### 포함 범위

- 서버 공개키 조회
- 요청 단위 AES 키 생성
- AES 키의 RSA 공개키 암호화
- 프롬프트 및 문의 데이터의 AES 암호화
- 암호화 요청 Envelope 구성
- 암호화 응답 복호화
- 키와 평문 데이터의 수명 관리
- 암복호화 실패 및 재시도 처리

### 제외 범위

- 채팅 UI 구성
- 대화 상태 머신
- LLM Prompt 설계
- 사용자 인증 및 권한 정책
- 서버 내부의 LLM 호출 방식
- 스트리밍 응답 처리

---

## 3. 목표 구조

```text
User Input
    ↓
Plain Request Model
    ↓
Encryption Coordinator
    ├─ Public Key Provider
    ├─ AES Key Generator
    ├─ Payload Encryptor
    └─ Request Envelope Builder
    ↓
API Client
    ↓
Encrypted Response Envelope
    ↓
Response Decryptor
    ↓
Plain Response Model
    ↓
Application State
```

프론트엔드는 암호화 세부 구현을 UI 및 채팅 상태 관리 계층에서 분리한다. 상위 계층은 평문 요청 모델과 평문 응답 모델만 취급하며, 네트워크 경계에서만 암호화 Envelope로 변환한다.

---

## 4. 주요 컴포넌트

### 4.1 Public Key Provider

서버의 공개키를 조회하고, 사용 가능한 형식으로 변환하여 암호화 계층에 제공한다.

**책임**

- 공개키 API 호출
- 공개키 형식 및 알고리즘 식별자 검증
- 만료 정보 또는 키 식별자 관리
- 메모리 캐시 적용
- 키 교체 감지 시 캐시 무효화

**입력**

```text
Public Key 조회 요청
```

**출력**

```text
keyId
algorithm
publicKey
expiresAt(optional)
```

공개키는 비밀정보가 아니므로 브라우저에 제공할 수 있다. 다만 잘못된 공개키가 주입되지 않도록 HTTPS, 동일 출처 정책 및 응답 무결성 검증을 함께 사용해야 한다.

### 4.2 AES Key Generator

각 암호화 요청에 사용할 일회성 대칭키와 IV 또는 Nonce를 생성한다.

**설계 원칙**

- 브라우저의 암호학적 난수 생성기를 사용한다.
- 요청마다 새로운 AES 키를 생성한다.
- IV 또는 Nonce를 재사용하지 않는다.
- 키는 영구 저장소에 기록하지 않는다.
- 암호화 요청 완료 후 참조를 제거한다.

### 4.3 Payload Encryptor

평문 요청 객체를 직렬화하고 AES로 암호화한다.

**처리 순서**

```text
Plain Request
    ↓
Canonical Serialization
    ↓
UTF-8 Encoding
    ↓
AES Encryption
    ↓
Ciphertext + IV/Nonce + Authentication Tag
```

인증 암호화 모드를 사용하여 데이터 기밀성과 변조 탐지를 동시에 제공하는 것을 원칙으로 한다.

### 4.4 Key Wrapper

요청용 AES 키를 서버의 RSA 공개키로 암호화한다.

```text
AES Key
    ↓
RSA Public-Key Encryption
    ↓
Encrypted AES Key
```

RSA는 대용량 메시지 암호화가 아니라 대칭키 전달에만 사용한다.

### 4.5 Request Envelope Builder

암호화 결과를 네트워크 전송 계약에 맞게 조합한다.

권장 논리 구조는 다음과 같다.

```json
{
  "version": "1",
  "keyId": "server-key-id",
  "keyAlgorithm": "RSA",
  "contentAlgorithm": "AES",
  "encryptedKey": "base64...",
  "iv": "base64...",
  "ciphertext": "base64...",
  "tag": "base64..."
}
```

필드명과 인코딩 방식은 FE와 BE가 동일한 버전 계약으로 관리한다.

### 4.6 Response Decryptor

서버가 동일 세션 키로 암호화해 반환한 응답을 복호화한다.

**처리 순서**

```text
Encrypted Response Envelope
    ↓
Schema Validation
    ↓
Base64 Decode
    ↓
AES Decryption and Integrity Check
    ↓
UTF-8 Decode
    ↓
JSON Parse
    ↓
Plain Response Model
```

복호화 전에 반드시 버전, 알고리즘, 필수 필드 및 데이터 크기를 검증한다.

### 4.7 Encryption Coordinator

암호화 전체 흐름을 조정하는 단일 진입점이다.

상위 API Client는 다음과 같은 추상 인터페이스만 사용한다.

```text
encryptRequest(plainRequest)
decryptResponse(encryptedResponse, sessionContext)
```

UI와 채팅 상태 계층은 RSA, AES, IV, Base64와 같은 암호화 세부사항을 알지 못하도록 한다.

---

## 5. 요청 처리 시퀀스

```text
Chat State
    ↓ plain request
Encryption Coordinator
    ↓
Public Key Provider ── GET /public-key ──→ Backend
    ↓ public key
AES Key Generator
    ↓ session key + IV
Payload Encryptor
    ↓ ciphertext
Key Wrapper
    ↓ encrypted AES key
Envelope Builder
    ↓ encrypted request
API Client ── POST /chat ──→ Backend
```

공개키가 유효한 캐시에 존재하면 공개키 조회 단계는 생략할 수 있다.

---

## 6. 응답 처리 시퀀스

```text
Backend
    ↓ encrypted response
API Client
    ↓
Envelope Validator
    ↓
Response Decryptor
    ↓ plain response
Response Normalizer
    ↓
Chat State / UI
```

복호화가 성공한 데이터만 애플리케이션 상태에 반영한다. 복호화 또는 무결성 검증이 실패한 응답은 부분적으로 사용하지 않는다.

---

## 7. 상태 및 키 수명 관리

프론트엔드는 요청별로 다음 세션 문맥을 일시적으로 유지한다.

```text
requestId
keyId
AES key
request IV/Nonce
response decryption metadata
createdAt
```

### 수명 원칙

- AES 키는 해당 요청과 응답 복호화가 끝날 때까지만 메모리에 유지한다.
- LocalStorage, SessionStorage, IndexedDB에 AES 키를 저장하지 않는다.
- 평문 프롬프트와 복호화 응답을 디버그 로그에 기록하지 않는다.
- 요청 취소, 타임아웃, 오류 발생 시 세션 문맥을 폐기한다.
- 브라우저 새로고침 이후 키를 복구하지 않는다.

---

## 8. 오류 처리 정책

| 오류 유형 | FE 처리 원칙 |
|---|---|
| 공개키 조회 실패 | 요청 전송을 중단하고 재시도 가능한 오류로 분류 |
| 지원하지 않는 키 버전 | 공개키 캐시를 비우고 1회 재조회 |
| AES 키 생성 실패 | 요청을 전송하지 않고 보안 오류 반환 |
| 암호화 실패 | 평문 대체 전송 금지 |
| 네트워크 실패 | 동일 암호문 재전송 여부를 요청 멱등성 정책에 따라 결정 |
| 서버의 키 불일치 응답 | 최신 공개키를 다시 받은 후 새 AES 키로 요청 전체를 재암호화 |
| 응답 복호화 실패 | 응답 폐기, UI에 일반화된 오류 표시 |
| 무결성 검증 실패 | 보안 오류로 기록하고 데이터 사용 금지 |

암호화 실패 시 평문 요청으로 자동 전환하는 Fallback은 허용하지 않는다.

---

## 9. API 계약

### 9.1 공개키 조회

```text
GET /api/public-key
```

**응답 논리 모델**

```text
version
keyId
algorithm
publicKey
expiresAt(optional)
```

### 9.2 암호화 채팅 요청

```text
POST /api/chat
Content-Type: application/json
```

**요청 논리 모델**

```text
version
requestId
keyId
encryptedKey
iv/nonce
ciphertext
tag
```

**응답 논리 모델**

```text
version
requestId
iv/nonce
ciphertext
tag
```

요청과 응답의 `requestId`를 연결하여 다른 요청의 암호화 응답이 잘못 적용되는 것을 방지한다.

---

## 10. 보안 설계 원칙

- HTTPS를 필수로 사용한다.
- 브라우저 표준 암호화 API를 사용한다.
- 인증 암호화 모드를 사용한다.
- RSA는 검증된 패딩 방식을 사용하고 AES 키 포장에만 사용한다.
- 요청별 AES 키와 IV/Nonce를 재사용하지 않는다.
- 알고리즘과 키 버전을 Envelope에 명시한다.
- 평문 및 키를 로그, 오류 추적 도구, 분석 이벤트에 포함하지 않는다.
- 암호화 기능이 실패한 경우 Fail Closed 방식으로 처리한다.
- 암호화는 XSS로부터 평문을 보호하지 못하므로 CSP, 입력 검증 및 의존성 보안이 별도로 필요하다.

---

## 11. 비기능 요구사항

### 성능

- 공개키는 유효기간 동안 메모리에 캐시한다.
- 암복호화는 UI 스레드 정지를 최소화하도록 비동기로 수행한다.
- 대용량 요청은 사전에 크기를 제한한다.

### 호환성

- 지원 브라우저의 암호화 API 제공 여부를 초기화 단계에서 확인한다.
- 암호화 미지원 브라우저에서는 기능을 비활성화하며 평문 전송으로 대체하지 않는다.

### 관측성

다음 메타데이터만 기록할 수 있다.

```text
requestId
keyId
algorithm version
encryption duration
decryption duration
error category
```

평문, 대칭키, 전체 암호문은 원칙적으로 로그에 남기지 않는다.

---

## 12. 테스트 설계

### 단위 테스트

- 공개키 응답 검증
- 요청마다 새로운 AES 키와 IV 생성 여부
- 직렬화 및 역직렬화 일관성
- 정상 암복호화
- 암호문, 태그, IV 변조 시 실패 여부
- 잘못된 키 버전 처리

### 통합 테스트

- 공개키 조회부터 응답 복호화까지 전체 흐름
- 서버 키 교체 직후 재시도
- 네트워크 중단 및 요청 취소
- 복수 요청 동시 처리 시 키 문맥 분리

### 보안 테스트

- 평문 로그 노출 여부
- 저장소에 키가 남지 않는지 확인
- 암호문 재전송 및 요청 식별자 충돌 검증
- XSS 상황에서 암호화만으로 보호된다고 잘못 가정하지 않는지 검증

---

## 13. 완료 기준

- UI 및 상태 계층이 암호화 구현에 직접 의존하지 않는다.
- 모든 채팅 요청은 암호화 Envelope로 전송된다.
- 모든 채팅 응답은 무결성 검증 후 복호화된다.
- 키 불일치 시 최신 공개키를 사용하여 안전하게 재시도한다.
- 오류 발생 시 평문 Fallback이 수행되지 않는다.
- 평문과 대칭키가 브라우저 영구 저장소 및 운영 로그에 남지 않는다.
