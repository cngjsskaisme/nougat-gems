# Backend Structured UI Response Layer

## 구현 대상

LLM의 응답을 구조화된 UI 명세로 생성하고 검증하여  
Frontend에 전달하는 Backend 계층을 구현한다.

Backend는 LLM이 반환한 값을 직접 실행하지 않으며,  
사전에 정의된 Component, Props, Action만 허용한다.

# 구현 목적

## 구조화된 응답 생성

LLM이 일반 텍스트나 임의 JSON이 아니라  
정해진 UI Contract에 맞는 응답을 생성하도록 한다.

## UI 코드 생성 방지

LLM이 다음 내용을 직접 생성하지 못하도록 한다.

- HTML
- JavaScript
- Vue Template
- 실행 가능한 함수
- 임의 API 호출 코드
- 임의 Component 이름

## Backend 검증

LLM이 Structured Output을 반환하더라도  
Backend에서 별도의 Runtime Validation과 Business Rule Validation을 수행한다.

## 실제 작업 실행 분리

LLM은 사용자가 수행할 수 있는 Action을 제안할 수 있지만,  
문의 전송이나 외부 API 호출을 직접 실행하지 않는다.

# 전체 처리 흐름

## Chat 요청 처리

Client Request 수신

→ Request 형식 검증

→ 암호화 Payload 복호화

→ Conversation Context 구성

→ System Rules 결합

→ LLM Structured Output 요청

→ JSON Schema 검증

→ Component 검증

→ Props 검증

→ Action 검증

→ Inquiry State 검증

→ Metadata 생성

→ 응답 암호화

→ Frontend 반환

# 응답 Contract

## 공통 응답 구조

모든 LLM UI 응답은 다음 항목으로 구성한다.

- version
- component
- props
- actions
- metadata

## version

FE와 BE가 사용하는 UI Contract의 버전이다.

용도는 다음과 같다.

- 응답 형식 호환성 확인
- 이전 버전 변환
- 지원하지 않는 버전 차단
- Component 및 Props 변경 관리

## component

Frontend에서 렌더링해야 할 UI 종류다.

component 값은 사전에 정의된 Enum 중 하나만 허용한다.

## props

선택된 Component에 전달할 표시 데이터다.

각 Component는 별도의 Props Schema를 가진다.

## actions

사용자가 선택할 수 있는 행동 목록이다.

Action은 실제 함수나 실행 코드를 포함하지 않고  
선언적인 Action Type과 Payload만 포함한다.

## metadata

Backend가 생성하는 요청 추적 정보다.

Metadata에는 다음 정보를 포함할 수 있다.

- requestId
- createdAt
- model
- provider
- contractVersion

Metadata는 LLM이 아니라 Backend가 생성한다.

# 허용 Component

## markdown

일반적인 텍스트 또는 Markdown 답변을 표시한다.

### 주요 Props

- content

### 허용 Action

- 없음
- 필요 시 startInquiry
- 필요 시 openRegisteredLink

## inquiryDraftCard

사용자가 문의 내용을 입력하거나 수정할 수 있는 카드를 표시한다.

### 주요 Props

- title
- description
- fields
- validationMessage

### 허용 Action

- updateInquiryDraft
- requestInquiryConfirmation
- cancelInquiry

## inquiryConfirmCard

작성된 문의 내용을 사용자에게 최종 확인시키는 카드를 표시한다.

### 주요 Props

- title
- summary
- notice

### 허용 Action

- editInquiry
- sendInquiry

## inquirySuccessCard

문의가 실제로 전달된 이후 성공 결과를 표시한다.

### 주요 Props

- title
- message
- referenceId
- processedAt

### 허용 Action

- 없음
- 필요 시 startNewInquiry

### 생성 원칙

이 Component는 LLM의 판단만으로 생성하지 않는다.

실제 문의 전송 API가 성공한 이후  
Backend 또는 FE가 신뢰 가능한 결과를 기반으로 생성한다.

## errorCard

응답 처리 또는 외부 작업 중 오류가 발생했을 때 표시한다.

### 주요 Props

- title
- message
- retryable
- errorCode

### 허용 Action

- retryChat
- retryInquiry
- cancelInquiry

# 허용 Action

## startInquiry

문의 작성 흐름을 시작한다.

## updateInquiryDraft

사용자가 입력한 문의 초안을 Client State에 반영한다.

## requestInquiryConfirmation

문의 필수값을 검증하고 확인 단계로 이동한다.

## editInquiry

확인 단계에서 다시 작성 단계로 이동한다.

## sendInquiry

사용자가 확인한 문의를 실제 전송 API로 전달하도록 요청한다.

## retryChat

직전 Chat 요청을 다시 수행한다.

## retryInquiry

실패한 문의 전송을 다시 수행한다.

## cancelInquiry

문의 작성 흐름을 종료하고 초기 상태로 돌아간다.

## openRegisteredLink

사전에 등록된 Link ID에 해당하는 외부 또는 내부 링크를 연다.

LLM이 임의 URL을 직접 전달하는 방식은 기본적으로 사용하지 않는다.

# Action 공통 구조

## Action ID

한 응답 안에서 Action을 식별하기 위한 고유 값이다.

## Action Type

사전에 정의된 Action Enum 중 하나다.

## Label

버튼이나 메뉴에 표시할 사용자용 문구다.

## Style

UI 표현 목적의 제한된 값이다.

예시는 다음과 같다.

- primary
- secondary
- danger
- text

## Payload

Action 실행에 필요한 제한적인 데이터다.

Payload에는 다음을 포함하지 않는다.

- 실행 코드
- 함수명
- Script
- Component 경로
- 임의 API Endpoint
- 인증 정보

# Component별 Props 검증

## markdown 검증

다음 항목을 검증한다.

- content가 문자열인지
- 빈 문자열인지
- 최대 길이를 초과했는지
- Raw HTML 허용 정책에 위배되는지
- 위험한 링크 형식이 포함되었는지

## inquiryDraftCard 검증

다음 항목을 검증한다.

- title 존재 여부
- description 타입
- fields 배열 여부
- 허용된 Field 이름인지
- 허용된 Input Type인지
- required 값이 Boolean인지
- initialValue 최대 길이
- 중복 Field 존재 여부

## inquiryConfirmCard 검증

다음 항목을 검증한다.

- title 존재 여부
- summary 구조
- 필수 문의 정보 존재 여부
- 이메일 형식
- message 최대 길이
- notice 타입

## inquirySuccessCard 검증

다음 항목을 검증한다.

- 실제 전송 결과와 연결되어 있는지
- referenceId 형식
- processedAt 형식
- 전송 완료 이전에 생성되지 않았는지

## errorCard 검증

다음 항목을 검증한다.

- 사용자용 메시지인지
- 내부 예외 정보가 포함되지 않았는지
- retryable 값이 Boolean인지
- Error Type과 Retry Action이 일치하는지

# JSON Schema 정책

## Schema 기반 제한

다음 항목을 Schema로 정의한다.

- 필수 필드
- 데이터 타입
- Enum
- 배열 구조
- 최대 및 최소 개수
- 문자열 형식
- 허용되지 않은 추가 필드
- Component별 Props 구조
- Action별 Payload 구조

## 추가 필드 제한

정의되지 않은 필드가 응답에 포함되지 않도록  
가능하면 추가 속성을 허용하지 않는 정책을 사용한다.

## Schema 단순화

LLM Provider별 지원 범위가 다를 수 있으므로  
공통 Schema는 다음 요소 중심으로 구성한다.

- object
- array
- string
- number
- boolean
- enum
- required
- anyOf

# Backend Runtime Validation

## Validation 시점

LLM 응답을 수신한 직후 수행한다.

## Validation 항목

- JSON Parse 가능 여부
- 공통 Contract 검증
- Contract Version 검증
- Component Enum 검증
- Component별 Props 검증
- Action Enum 검증
- Action별 Payload 검증
- 응답 크기 검증
- 문자열 길이 검증

## Validation 실패 처리

검증에 실패한 LLM 원본 응답을 FE에 전달하지 않는다.

대신 Backend가 생성한 안전한 errorCard 응답을 반환한다.

# Component-Action 정책 검증

## 검증 목적

Schema상 유효한 Action이라도  
해당 Component에서 사용할 수 있는지 별도로 검증한다.

## 허용 관계

### markdown

- startInquiry
- openRegisteredLink

### inquiryDraftCard

- updateInquiryDraft
- requestInquiryConfirmation
- cancelInquiry

### inquiryConfirmCard

- editInquiry
- sendInquiry

### inquirySuccessCard

- startNewInquiry

### errorCard

- retryChat
- retryInquiry
- cancelInquiry

## 정책 위반 처리

허용되지 않은 Component-Action 조합은  
Structured Response 오류로 처리한다.

# Inquiry State 정책

## 상태 목록

- idle
- editing
- readyToSend
- sending
- sent
- error

## 상태별 허용 Component

### idle

- markdown
- inquiryDraftCard

### editing

- inquiryDraftCard
- errorCard

### readyToSend

- inquiryConfirmCard
- inquiryDraftCard

### sending

- 별도 Loading 상태
- errorCard

### sent

- inquirySuccessCard
- markdown

### error

- errorCard
- inquiryDraftCard
- inquiryConfirmCard

## 상태 검증 원칙

- 필수 정보 없이 readyToSend 상태로 이동할 수 없다.
- idle 상태에서 바로 sending 상태로 이동할 수 없다.
- sending 상태에서 중복 sendInquiry를 허용하지 않는다.
- sent 상태에서 같은 문의를 다시 전송하지 않는다.
- 실제 API 성공 전에는 sent 상태를 반환하지 않는다.

# Prompt Builder 정책

## Prompt 구성

Prompt는 다음 순서로 구성한다.

- System Rules
- UI Contract
- Component 정의
- Action 정의
- State 정보
- Portfolio 또는 Business Context
- Conversation
- Output Schema

## 사용자 입력 분리

System Instruction과 사용자 Conversation을 명확히 구분한다.

## Prompt Injection 대응

사용자 입력에 다음과 같은 요청이 포함되어도 따르지 않는다.

- 기존 Schema 무시
- 임의 Component 생성
- sendInquiry 즉시 실행
- System Prompt 공개
- HTML 또는 Script 반환
- 등록되지 않은 Action 생성
- 내부 API 호출

# Response Normalization

## 기본값 통일

선택 필드의 기본값을 일관되게 처리한다.

예시는 다음과 같다.

- 값이 없으면 null
- 목록이 없으면 빈 배열
- Action이 없으면 빈 배열
- 선택 문자열이 없으면 null

## Metadata 추가

Backend가 다음 값을 직접 추가한다.

- requestId
- createdAt
- model
- provider
- contractVersion

## Enum 표준화

Component와 Action 이름의 대소문자 및 형식을 통일한다.

# 실제 문의 전송 정책

## LLM 역할

LLM은 사용자가 문의를 보낼 의도가 있는지 판단하고  
inquiryConfirmCard 및 sendInquiry Action을 제안할 수 있다.

## 사용자 역할

사용자가 전송 버튼을 명시적으로 선택해야 한다.

## Frontend 역할

Frontend는 사용자의 Action을 받아  
별도의 send-message API를 호출한다.

## Backend 역할

Backend는 문의 내용을 다시 검증한 후  
Telegram API 등 실제 외부 서비스를 호출한다.

# 문의 전송 API 입력

## 필수 입력

- name
- email
- message
- idempotencyKey

## 선택 입력

- organization
- phone
- inquiryCategory
- sourcePage

# 문의 전송 API 검증

## 입력 검증

- 필수값 존재 여부
- 이메일 형식
- 문자열 최대 길이
- 금지된 Content 형식
- Request 크기
- 암호화 세션 유효성

## 실행 검증

- 현재 State가 readyToSend인지
- 동일 idempotencyKey가 처리되었는지
- Rate Limit을 초과했는지
- Bot 또는 Abuse 가능성이 있는지

# 문의 전송 API 처리

## 처리 순서

Request 수신

→ 복호화

→ 입력값 검증

→ Idempotency 검증

→ Telegram용 메시지 생성

→ Telegram API 호출

→ 실제 성공 여부 확인

→ 처리 결과 생성

→ 응답 암호화

→ Frontend 반환

# 문의 전송 API 출력

## 성공 응답

다음 정보를 반환한다.

- success
- referenceId
- processedAt

## 실패 응답

다음 정보를 반환한다.

- success
- retryable
- errorCode
- requestId

내부 Telegram 오류나 Stack Trace는 반환하지 않는다.

# Stateless 정책

## Chat API

일반 Chat API는 Stateless를 유지한다.

Backend는 다음 정보를 영구 저장하지 않는다.

- 전체 Conversation
- 일반 Chat Message
- 현재 UI Component
- Inquiry 작성 단계
- 문의 Draft

## Client Context

각 요청에서 FE가 필요한 Conversation과 State를 전달한다.

## 예외적인 단기 상태

중복 실행 방지를 위해 다음 정보는 짧은 기간 저장할 수 있다.

- idempotencyKey
- 전송 처리 상태
- Rate Limit 정보
- 요청 Fingerprint

이는 Conversation Memory가 아니라  
외부 작업의 중복 실행을 방지하기 위한 운영 상태다.

# 오류 처리

## 오류 유형

- requestValidationError
- decryptionError
- llmProviderError
- structuredOutputError
- componentValidationError
- propsValidationError
- actionValidationError
- statePolicyError
- sendInquiryError
- encryptionError

## 사용자 응답

Frontend에는 다음 정보만 전달한다.

- 사용자용 메시지
- 재시도 가능 여부
- requestId
- 제한된 errorCode

## 내부 처리

다음 정보는 사용자에게 노출하지 않는다.

- Stack Trace
- System Prompt
- API Key
- 복호화 Key
- 내부 Endpoint
- LLM 원본 응답 전체

# 보안 정책

## Allowlist

다음 항목은 모두 Allowlist 방식으로 관리한다.

- Component
- Action
- Link
- Field
- Input Type
- Action Payload
- 외부 API 기능

## 임의 코드 실행 차단

LLM 응답을 다음 방식으로 사용하지 않는다.

- eval
- Function Constructor
- 동적 Import 경로
- 함수명 직접 실행
- Template 직접 컴파일
- 임의 Endpoint 호출

## URL 보안

외부 링크가 필요한 경우 URL 대신 Link ID 사용을 권장한다.

Backend 또는 FE의 Link Registry가  
Link ID를 실제 URL과 연결한다.

## 개인정보 보호

다음 정보는 운영 로그에 평문으로 기록하지 않는다.

- 문의 내용
- 이름
- 이메일
- 전화번호
- 암호화 Key
- 복호화된 전체 Payload

# 로깅 및 관측

## 기록 항목

- requestId
- contractVersion
- component
- action type
- Validation 결과
- State Policy 결과
- 처리 시간
- LLM Provider
- Error Category

## 기록 제외 항목

- Prompt 전문
- Conversation 전문
- 문의 전문
- 개인정보
- 암호화 Key
- 전체 복호화 Payload

# Backend 완료 조건

## Contract 완료 조건

- 모든 응답이 정의된 Contract를 따른다.
- 임의 Component가 FE로 전달되지 않는다.
- 임의 Action이 FE로 전달되지 않는다.
- Component별 Props가 검증된다.
- Contract Version이 관리된다.

## 실행 안전성 완료 조건

- LLM 응답만으로 외부 작업이 실행되지 않는다.
- 사용자의 명시적 Action 이후에만 문의가 전송된다.
- 문의 전송은 별도 API에서 다시 검증된다.
- 중복 전송이 방지된다.

## 오류 처리 완료 조건

- Schema 오류 시 안전한 errorCard가 반환된다.
- 내부 예외 정보가 노출되지 않는다.
- requestId로 오류 추적이 가능하다.