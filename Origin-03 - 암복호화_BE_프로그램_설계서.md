# 암복호화 BE 프로그램 설계서

## 1. 문서 목적

본 문서는 채팅 백엔드가 브라우저에서 전달된 암호화 요청을 복호화하고, 처리 결과를 다시 암호화하여 반환하기 위한 서버 측 설계를 정의한다.

보호 구간은 다음과 같다.

```text
Browser ↔ Chat Backend
```

백엔드는 RSA 키 쌍을 관리하고, 브라우저가 RSA 공개키로 감싼 요청별 AES 키를 개인키로 복호화한다. 실제 프롬프트와 응답 데이터는 AES로 암복호화한다.

---

## 2. 설계 범위

### 포함 범위

- RSA 키 쌍 생성 또는 안전한 로딩
- 공개키 제공 API
- 키 식별자와 키 버전 관리
- 암호화 요청 Envelope 검증
- RSA 기반 AES 키 복호화
- AES 기반 요청 데이터 복호화
- 평문 요청의 내부 서비스 전달
- 처리 결과의 AES 암호화
- 오류 응답과 감사 로그 정책

### 제외 범위

- 프론트엔드 UI 및 상태 관리
- LLM Prompt 내용
- 채팅 업무 규칙
- Telegram 등 외부 전달 기능
- 스트리밍 응답
- 사용자 계정 인증 정책

---

## 3. 목표 구조

```text
HTTP Router
    ↓
Encrypted Request Validator
    ↓
Key Resolver
    ↓
RSA Key Unwrapper
    ↓
AES Request Decryptor
    ↓
Plain Chat Request
    ↓
Chat / LLM Service
    ↓
Plain Chat Response
    ↓
AES Response Encryptor
    ↓
Encrypted Response Builder
    ↓
HTTP Response
```

암복호화 계층은 HTTP 라우팅과 채팅 업무 로직 사이의 독립된 보안 경계로 둔다. 채팅 서비스는 암호화 Envelope를 알지 못하고 평문 도메인 모델만 처리한다.

---

## 4. 주요 컴포넌트

### 4.1 Key Manager

RSA 키 쌍과 키 메타데이터를 관리한다.

**책임**

- 서버 시작 시 키 로딩 또는 생성
- 활성 키와 이전 키 관리
- `keyId` 기반 키 조회
- 공개키 직렬화
- 키 교체와 폐기
- 개인키 접근 권한 통제

**권장 운영 원칙**

- 개발 환경의 임시 키 생성과 운영 환경의 영속 키 관리를 분리한다.
- 운영 개인키는 소스 코드와 일반 환경변수에 직접 포함하지 않는다.
- 다중 인스턴스 환경에서는 모든 인스턴스가 동일한 활성 키 집합을 사용한다.
- 키 교체 중에는 짧은 유예기간 동안 이전 개인키를 함께 유지한다.

서버 시작마다 새로운 키를 생성하면 재시작 시 기존 공개키를 사용한 진행 중 요청이 실패할 수 있으므로, 운영 환경에서는 키 저장 및 회전 전략이 필요하다.

### 4.2 Public Key Controller

클라이언트가 요청 암호화에 사용할 공개키와 메타데이터를 제공한다.

```text
GET /public-key
```

**응답 논리 모델**

```text
version
keyId
algorithm
publicKey
expiresAt(optional)
```

공개키 응답은 캐시할 수 있지만, 키 교체 주기와 만료 정책에 맞는 Cache-Control을 적용한다.

### 4.3 Encrypted Request Validator

암호화 Envelope의 구조와 크기를 검증한다.

**검증 항목**

- 지원하는 계약 버전인지 여부
- `keyId` 존재 및 형식
- 지원하는 RSA/AES 알고리즘인지 여부
- `encryptedKey`, `iv/nonce`, `ciphertext`, `tag` 존재 여부
- Base64 또는 합의된 인코딩 유효성
- 허용된 최대 요청 크기
- `requestId` 형식과 중복 여부

검증 실패 요청은 복호화를 시도하지 않고 즉시 거부한다.

### 4.4 RSA Key Unwrapper

요청 Envelope의 암호화된 AES 키를 개인키로 복호화한다.

```text
Encrypted AES Key
    ↓
Resolve Private Key by keyId
    ↓
RSA Private-Key Decryption
    ↓
AES Session Key
```

복호화 실패의 상세 원인은 외부 응답에 노출하지 않는다.

### 4.5 AES Request Decryptor

AES 키와 IV/Nonce를 이용해 요청 본문을 복호화하고 무결성을 확인한다.

```text
Ciphertext + IV/Nonce + Tag
    ↓
AES Decryption and Integrity Verification
    ↓
UTF-8 Plaintext
    ↓
JSON Parse
    ↓
Plain Request Model
```

무결성 검증 실패 시 평문 데이터는 생성하거나 업무 계층으로 전달하지 않는다.

### 4.6 Secure Request Context

한 요청에서만 사용하는 보안 문맥이다.

```text
requestId
keyId
AES session key
request cryptographic metadata
createdAt
```

이 문맥은 요청 처리 완료, 취소 또는 오류 시 즉시 폐기한다. DB, 캐시, 메시지 큐에 AES 키를 저장하지 않는다.

### 4.7 AES Response Encryptor

업무 처리 결과를 요청에서 복원한 AES 키로 암호화한다.

```text
Plain Response
    ↓
Canonical Serialization
    ↓
New Response IV/Nonce
    ↓
AES Encryption
    ↓
Ciphertext + Tag
```

응답에는 요청과 다른 IV/Nonce를 사용한다. 동일 키를 사용하더라도 IV/Nonce는 절대 재사용하지 않는다.

### 4.8 Encrypted Response Builder

암호화 응답을 계약 형식으로 구성한다.

```json
{
  "version": "1",
  "requestId": "request-id",
  "iv": "base64...",
  "ciphertext": "base64...",
  "tag": "base64..."
}
```

응답은 원 요청의 `requestId`와 연결한다.

---

## 5. 공개키 제공 시퀀스

```text
Browser
    ↓ GET /public-key
Public Key Controller
    ↓
Key Manager
    ↓ active public key + metadata
Public Key Controller
    ↓
Browser
```

개인키는 어떤 응답에도 포함하지 않는다.

---

## 6. 채팅 요청 처리 시퀀스

```text
Browser
    ↓ encrypted request
HTTP Router
    ↓
Envelope Validator
    ↓
Key Manager ── resolve by keyId
    ↓
RSA Key Unwrapper
    ↓ AES session key
AES Request Decryptor
    ↓ plain request
Chat / LLM Service
    ↓ plain response
AES Response Encryptor
    ↓ encrypted response
HTTP Router
    ↓
Browser
```

암호화 계층과 채팅 서비스 사이에는 평문 도메인 DTO를 사용하되, 평문이 로그나 예외 메시지로 유출되지 않도록 한다.

---

## 7. 키 관리 정책

### 활성 키

새로운 클라이언트 요청에 공개되는 현재 키이다.

### 이전 키

키 교체 직전에 공개되었던 키로, 진행 중인 요청 처리를 위해 제한된 유예기간 동안 복호화에만 사용한다.

### 폐기 키

유예기간이 끝난 키이며 서버 메모리와 키 저장소에서 안전하게 제거한다.

### 키 교체 흐름

```text
Create New Key Pair
    ↓
Register New keyId
    ↓
Publish New Public Key
    ↓
Keep Previous Private Key Temporarily
    ↓
Expire Previous keyId
    ↓
Securely Remove Previous Private Key
```

키 교체는 무중단 배포와 다중 인스턴스 동기화를 고려해야 한다.

---

## 8. 오류 처리 정책

| 오류 유형 | 서버 처리 | 외부 응답 원칙 |
|---|---|---|
| 잘못된 Envelope | 복호화 전 거부 | 일반화된 요청 오류 |
| 알 수 없는 `keyId` | 최신 공개키 재조회 필요 상태 반환 | 내부 키 정보 비노출 |
| RSA 복호화 실패 | 요청 중단 및 보안 이벤트 기록 | 일반화된 복호화 오류 |
| AES 무결성 실패 | 평문 폐기 | 일반화된 복호화 오류 |
| 평문 JSON 검증 실패 | 업무 계층 전달 금지 | 유효하지 않은 요청 |
| LLM 처리 실패 | 오류 응답 모델 생성 후 필요 시 암호화 | 내부 공급자 정보 비노출 |
| 응답 암호화 실패 | 평문 응답 반환 금지 | 서버 오류 |
| 요청 취소 | 키 문맥 제거, 후속 처리 중단 | 연결 종료 또는 취소 응답 |

암복호화 실패의 세부 원인, 개인키 존재 여부, 패딩 오류 등은 공격자에게 노출하지 않는다.

---

## 9. API 계약

### 9.1 공개키 API

```text
GET /public-key
```

**성공 응답**

```text
version
keyId
algorithm
publicKey
expiresAt(optional)
```

### 9.2 채팅 API

```text
POST /chat
Content-Type: application/json
```

**암호화 요청**

```text
version
requestId
keyId
keyAlgorithm
contentAlgorithm
encryptedKey
iv/nonce
ciphertext
tag
```

**암호화 성공 응답**

```text
version
requestId
iv/nonce
ciphertext
tag
```

오류 응답도 민감한 내부 정보를 포함하지 않도록 별도 표준 오류 계약을 사용한다.

---

## 10. 보안 설계 원칙

- HTTPS를 필수로 사용한다.
- 인증 암호화 모드를 사용한다.
- RSA 개인키 접근을 암복호화 모듈로 제한한다.
- RSA는 AES 키 복호화에만 사용한다.
- 요청과 응답에 서로 다른 IV/Nonce를 사용한다.
- 키와 평문을 운영 로그, APM, 트레이스에 포함하지 않는다.
- 요청 크기 제한과 처리 시간 제한을 적용한다.
- 잘못된 암호문을 반복 제출하는 요청에 Rate Limit을 적용한다.
- 오류 메시지와 처리 시간을 통해 복호화 상세가 추론되지 않도록 한다.
- 암호화 계층 이후 LLM 공급자에게 데이터가 전달될 경우, 해당 구간은 별도의 보안·개인정보 정책으로 관리한다.

이 구조는 브라우저와 채팅 백엔드 사이를 보호하며, 백엔드가 평문을 처리하므로 백엔드 자체와 이후의 LLM 호출 구간까지 종단 간 비공개가 보장되는 구조는 아니다.

---

## 11. 관측성 및 감사 로그

기록 가능한 항목은 다음과 같다.

```text
requestId
keyId
contract version
ciphertext size
decryption duration
encryption duration
result status
error category
client metadata(minimized)
```

기록 금지 항목은 다음과 같다.

```text
private key
AES session key
plaintext request
plaintext response
full encrypted key
full ciphertext
```

보안 이벤트는 일반 애플리케이션 로그와 분리하여 접근을 제한한다.

---

## 12. 비기능 요구사항

### 가용성

- 키 저장소 장애 시 새 요청을 안전하게 거부한다.
- 다중 인스턴스가 동일한 키 버전을 사용할 수 있어야 한다.
- 키 교체 중 이전 공개키 기반 요청을 제한적으로 처리할 수 있어야 한다.

### 성능

- RSA 연산은 요청당 AES 키 복호화 1회로 제한한다.
- 실제 메시지 처리는 AES를 사용한다.
- 암복호화 시간과 LLM 처리 시간을 분리하여 측정한다.

### 확장성

- 계약 버전과 알고리즘 식별자를 통해 향후 알고리즘 교체를 지원한다.
- 암복호화 모듈은 Chat Service와 독립적으로 배포 또는 교체할 수 있는 인터페이스를 유지한다.

---

## 13. 테스트 설계

### 단위 테스트

- 활성 키 조회와 이전 키 조회
- 정상 RSA 키 복호화
- 잘못된 `keyId` 처리
- 정상 AES 복호화 및 암호화
- 암호문, 태그, IV 변조 탐지
- 응답 IV/Nonce 재사용 방지

### 통합 테스트

- 공개키 조회부터 암호화 응답 반환까지 전체 흐름
- 키 교체 중 이전 키 요청 처리
- 서버 재시작 및 다중 인스턴스 환경
- LLM 오류 발생 시 암호화 오류 응답 정책
- 요청 취소 시 키 문맥 제거

### 보안 테스트

- Padding Oracle 및 오류 메시지 노출 점검
- 과대 암호문 요청에 대한 제한
- 반복 복호화 실패 요청에 대한 Rate Limit
- 로그와 트레이스의 평문 및 키 노출 여부
- 키 저장소 권한과 키 회전 절차 검증

---

## 14. 완료 기준

- 공개키와 개인키의 역할 및 수명이 분리되어 있다.
- 암호화 Envelope 검증 후에만 복호화를 수행한다.
- Chat Service는 평문 도메인 모델만 처리하고 암호화 구현을 알지 못한다.
- 모든 정상 응답은 AES로 암호화되어 반환된다.
- 응답 암호화 실패 시 평문 응답을 반환하지 않는다.
- 운영 환경에서 키 교체와 다중 인스턴스 동기화가 가능하다.
- 평문과 키가 로그 및 영속 저장소에 남지 않는다.
