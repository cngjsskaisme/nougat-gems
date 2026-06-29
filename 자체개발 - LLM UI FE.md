# Frontend Structured UI Rendering Layer

## 구현 대상

Backend가 반환한 Structured UI Response를 검증하고  
등록된 Vue Component로 렌더링하는 Frontend 계층을 구현한다.

사용자 Action은 Action Dispatcher와 State Machine을 통해 처리하며,  
UI Component는 직접 API를 호출하거나 상태를 변경하지 않는다.

# 구현 목적

## 동적 UI 렌더링

Backend의 component 값을 기준으로  
사전에 등록된 Vue Component를 선택하여 렌더링한다.

## 안전한 Component 선택

Backend에서 받은 문자열을  
Vue Component 이름이나 Import 경로로 직접 사용하지 않는다.

Component Registry에 등록된 값만 렌더링한다.

## UI와 Business Logic 분리

UI Component는 다음 역할만 담당한다.

- 데이터 표시
- 사용자 입력 표시
- 클릭 Event 발생
- 입력 Event 발생
- Loading 상태 표시

API 호출과 State 변경은 별도 계층이 담당한다.

# 전체 처리 흐름

## Chat 응답 처리

API Response 수신

→ AES 복호화

→ JSON Parse

→ Contract Version 확인

→ Runtime Schema 검증

→ Response Normalize

→ Message Store 저장

→ Component Registry 조회

→ Dynamic Component 렌더링

## 사용자 Action 처리

사용자 버튼 선택

→ UI Component가 Event Emit

→ Response Renderer가 Event 수신

→ Action Dispatcher 전달

→ Action Registry 조회

→ Action Payload 검증

→ State Machine Guard 확인

→ Handler 실행

→ State 또는 Message Store 변경

→ 필요 시 API 호출

# Frontend 계층 구성

## Response Parser

복호화된 응답을 JSON Object로 변환한다.

## Response Validator

응답의 Contract와 데이터 구조를 검증한다.

## Response Normalizer

누락된 기본값과 이전 Contract Version을 정규화한다.

## Message Store

사용자와 Assistant의 Message를 저장한다.

## Component Registry

component 문자열을 실제 Vue Component와 연결한다.

## Response Renderer

선택된 Component를 렌더링하고 Action Event를 전달한다.

## Action Dispatcher

Action을 중앙 진입점에서 받아 검증하고 실행한다.

## Action Registry

Action Type을 FE 내부 Handler와 연결한다.

## Inquiry State Machine

문의 작성 및 전송 상태를 관리한다.

## API Client

다음 API 호출을 담당한다.

- public-key
- chat
- send-message

## Fallback UI

유효하지 않은 응답이나 렌더링 오류를 안전하게 표시한다.

# Message Store

## User Message

User Message는 다음 정보를 가진다.

- id
- role
- content
- createdAt
- status

## Assistant Message

Assistant Message는 다음 정보를 가진다.

- id
- role
- response
- component
- props
- actions
- metadata
- createdAt
- status

## Message Status

허용 상태는 다음과 같다.

- pending
- completed
- failed

## 저장 제외 정보

다음 정보는 Message Store에 저장하지 않는다.

- RSA Private Key
- AES Session Key
- 복호화 전 임시 Buffer
- 인증 Token 원문
- 불필요한 LLM Provider 원본 응답

# Response Runtime Validation

## 검증 시점

복호화와 JSON Parse 이후  
Message Store에 저장하기 전에 검증한다.

## 공통 검증 항목

- 응답이 Object인지
- version 존재 여부
- component 존재 여부
- props Object 여부
- actions Array 여부
- metadata 구조
- 최대 응답 크기

## Component 검증

- Component Registry 등록 여부
- Component별 Props Schema
- 필수 Props
- 문자열 최대 길이
- 배열 최대 개수
- 허용 Field 여부

## Action 검증

- Action Registry 등록 여부
- Action Type
- Label
- Style
- Payload 구조
- Component와 Action의 허용 관계

## 검증 실패 처리

유효하지 않은 응답은 Message Store에  
일반 Assistant UI로 저장하지 않는다.

대신 Frontend가 생성한 errorCard 응답으로 대체한다.

# Contract Version

## 동일 버전

정상적으로 검증하고 렌더링한다.

## 지원 가능한 이전 버전

Response Normalizer를 통해 현재 버전으로 변환한다.

## 지원하지 않는 이전 버전

렌더링을 중단하고 errorCard를 표시한다.

## 미래 버전

현재 FE가 이해하지 못하는 응답이므로 렌더링하지 않는다.

사용자에게 새 버전이 필요하다는 일반적인 오류만 표시한다.

# Component Registry

## Registry 목적

Backend의 component 문자열과  
실제 Vue Component를 안전하게 연결한다.

## Registry 등록 항목

초기 Component는 다음과 같다.

- markdown
- inquiryDraftCard
- inquiryConfirmCard
- inquirySuccessCard
- errorCard

## Registry 정책

- 명시적으로 Import된 Component만 등록한다.
- 미등록 Component는 렌더링하지 않는다.
- 미등록 값은 errorCard로 대체한다.
- 외부 문자열을 Import 경로로 사용하지 않는다.
- 전역 Component 검색에 의존하지 않는다.
- Component 이름을 API Contract의 일부로 관리한다.

## Registry 비책임

Component Registry는 다음을 수행하지 않는다.

- API 호출
- State 변경
- Action 실행
- Props 가공
- 사용자 입력 검증

# Response Renderer

## Renderer 책임

- Response의 component 값을 읽는다.
- Component Registry에서 실제 Component를 조회한다.
- Component에 props를 전달한다.
- Component에 actions를 전달한다.
- Component의 Action Event를 수신한다.
- Event를 Action Dispatcher로 전달한다.

## Renderer 비책임

- Chat API 호출
- 문의 전송
- State Machine 직접 변경
- Action Business Logic 실행
- LLM 응답 생성
- Props 내용 재해석

## Renderer Fallback

Component 조회에 실패하거나  
렌더링 중 오류가 발생하면 errorCard를 표시한다.

# UI Component 공통 책임

## 표시 책임

UI Component는 다음을 표시한다.

- 제목
- 본문
- 입력 Field
- Action Button
- Loading 상태
- Validation Message

## Event 책임

UI Component는 다음 Event를 상위로 전달한다.

- action
- input
- change
- submit
- cancel

## 비책임

UI Component는 다음을 직접 수행하지 않는다.

- API 호출
- Telegram 전송
- State Machine 변경
- Message Store 직접 조작
- 다른 Component 선택
- Backend 응답 검증
- Action Handler 선택

# markdown Component

## 표시 항목

- Markdown Content

## 렌더링 정책

- Raw HTML 기본 비활성화
- Script 차단
- Inline Event Handler 차단
- 위험한 URL Scheme 차단
- 외부 링크 보안 속성 적용
- 최대 Content 길이 제한

## Action

기본적으로 Action을 표시하지 않는다.

필요한 경우 사전에 허용된 Action만 표시한다.

# inquiryDraftCard Component

## 표시 항목

- title
- description
- 문의 입력 Field
- Field별 오류 메시지
- 확인 버튼
- 취소 버튼

## 입력 Field

허용 가능한 Field 예시는 다음과 같다.

- name
- email
- organization
- phone
- message

## 처리 원칙

입력값 변경은 Event로 전달한다.

Component가 Chat State나 Draft Store를 직접 변경하지 않는다.

# inquiryConfirmCard Component

## 표시 항목

- 이름
- 이메일
- 소속
- 연락처
- 문의 내용
- 개인정보 안내
- 수정 버튼
- 전송 버튼

## 전송 상태

sending 상태에서는 다음 정책을 적용한다.

- 전송 버튼 비활성화
- 중복 클릭 차단
- Loading Indicator 표시
- 필요 시 수정 버튼 비활성화

## Event

- editInquiry
- sendInquiry

# inquirySuccessCard Component

## 표시 조건

실제 send-message API가 성공한 이후에만 표시한다.

## 표시 항목

- 완료 제목
- 완료 메시지
- referenceId
- processedAt

## 금지 조건

LLM이 성공 응답을 생성했다는 이유만으로 표시하지 않는다.

# errorCard Component

## 표시 항목

- 오류 제목
- 사용자용 오류 설명
- 재시도 가능 여부
- 재시도 버튼
- 필요 시 requestId

## 비표시 항목

- Stack Trace
- API Key
- Prompt
- LLM 원본 응답
- 내부 Endpoint
- 암호화 정보

# Action Registry

## Registry 목적

Backend에서 전달된 Action Type을  
Frontend 내부 Handler와 연결한다.

## 등록 Action

- startInquiry
- updateInquiryDraft
- requestInquiryConfirmation
- editInquiry
- sendInquiry
- retryChat
- retryInquiry
- cancelInquiry
- openRegisteredLink

## Registry 정책

- 등록되지 않은 Action은 실행하지 않는다.
- 문자열을 함수명으로 직접 실행하지 않는다.
- 동적 코드 실행을 사용하지 않는다.
- Payload를 실행 전에 검증한다.
- State Machine Guard를 먼저 확인한다.
- 실행 결과를 Message Store 또는 State에 반영한다.

# Action Dispatcher

## Dispatcher 책임

- Action Event 수신
- Action Registry 조회
- Action 존재 여부 확인
- Payload 검증
- State 확인
- State Transition Guard 실행
- Handler 호출
- Action 실행 오류 처리

## Dispatcher 비책임

- UI 직접 렌더링
- Component 직접 선택
- LLM 응답 생성
- 암호화 알고리즘 구현

# Inquiry State Machine

## 상태 목록

- idle
- editing
- readyToSend
- sending
- sent
- error

## idle 상태

문의가 시작되지 않은 상태다.

### 허용 전이

- editing

## editing 상태

사용자가 문의 내용을 입력하거나 수정하는 상태다.

### 허용 전이

- readyToSend
- idle

## readyToSend 상태

필수 정보가 검증되고  
사용자 확인을 기다리는 상태다.

### 허용 전이

- editing
- sending

## sending 상태

send-message API가 처리 중인 상태다.

### 허용 전이

- sent
- error

## sent 상태

실제 문의 전송이 완료된 상태다.

### 허용 전이

- idle
- editing

## error 상태

문의 전송 또는 처리 과정에서 오류가 발생한 상태다.

### 허용 전이

- editing
- sending
- idle

# State Transition Guard

## 검증 항목

- 현재 상태
- 요청된 다음 상태
- 필수 Draft 존재 여부
- 요청 진행 여부
- 이전 전송 성공 여부
- 중복 Action 여부

## 차단 예시

- idle에서 sending으로 직접 이동
- 필수값 없이 readyToSend 이동
- sending 중 sendInquiry 재실행
- sent 상태에서 동일 문의 재전송
- error 원인을 해결하지 않고 sent로 이동

# Draft State

## 관리 항목

- name
- email
- organization
- phone
- message

## Draft 저장 위치

useChatState 또는 별도의 Inquiry Store에서 관리한다.

## Draft 정책

- 입력 중 Draft 유지
- API 오류가 발생해도 Draft 유지
- 전송 성공 후 초기화 여부 결정
- 민감한 Draft를 Local Storage에 자동 저장하지 않는 것을 기본으로 함
- 새로고침 복원이 필요하면 저장 범위와 만료 정책을 별도로 정의

# requestInquiryConfirmation 처리

## 처리 순서

현재 상태 확인

→ Draft 필수값 검증

→ 이메일 형식 검증

→ 문자열 길이 검증

→ 오류가 있으면 editing 유지

→ 오류가 없으면 readyToSend 전환

→ inquiryConfirmCard 표시

# sendInquiry 처리

## 사전 조건

- 현재 State가 readyToSend
- 필수 Draft 검증 완료
- 전송 중이 아님
- 동일 문의가 이미 전송되지 않음

## 처리 순서

idempotencyKey 생성

→ State를 sending으로 변경

→ 전송 Button 비활성화

→ send-message API 호출

→ 실제 응답 확인

## 성공 처리

- State를 sent로 변경
- inquirySuccessCard 추가
- referenceId 저장
- Draft 초기화 정책 적용

## 실패 처리

- State를 error로 변경
- Draft 유지
- errorCard 표시
- retryable에 따라 재시도 Action 표시

# API Client

## public-key API

RSA Public Key를 요청한다.

## chat API

암호화된 Conversation과 State를 Backend에 전달한다.

## send-message API

사용자가 최종 확인한 문의를 별도로 전송한다.

## API Client 공통 정책

- Timeout
- Abort 처리
- Request ID
- Error Normalize
- 재시도 정책
- 중복 요청 방지
- 암호화 및 복호화 실패 처리

# External Link 처리

## 권장 구조

LLM 또는 Backend는 URL 대신 Link ID를 전달한다.

Frontend는 Link Registry에서  
Link ID에 대응하는 실제 URL을 조회한다.

## 검증 항목

- 등록된 Link ID인지
- 허용 Protocol인지
- 허용 Host인지
- 상대경로인지 절대경로인지
- 새 창 정책
- noopener 및 noreferrer 적용 여부

## 차단 항목

- javascript Scheme
- data Scheme
- 임의 iframe
- 등록되지 않은 Domain
- LLM이 생성한 검증되지 않은 URL

# Markdown 보안

## Parser 정책

Raw HTML을 기본적으로 허용하지 않는다.

## Sanitization

HTML 변환 결과를 사용하는 경우  
Sanitizer를 통과한 결과만 렌더링한다.

## 링크 정책

외부 링크는 다음 정책을 적용한다.

- 허용 Protocol 확인
- target 정책 확인
- rel 보안 속성 적용
- 위험한 URL 차단

# Loading UX

## Chat 요청 중

- 사용자 입력 중복 제출 차단
- Pending Assistant Message 표시
- 요청 취소 가능 여부 결정
- 응답 완료 후 상태 해제

## 문의 전송 중

- 전송 버튼 비활성화
- Loading Indicator 표시
- 중복 전송 차단
- Draft 유지

## 응답 완료 후

- 새 Message 위치로 스크롤
- Screen Reader 안내
- 적절한 Focus 이동
- 모바일과 데스크톱 UI 동기화

# Fallback 정책

## Fallback 발생 조건

- 응답 복호화 실패
- JSON Parse 실패
- Contract Version 불일치
- Schema Validation 실패
- 미등록 Component
- 미등록 Action
- Props 누락
- Renderer 오류
- Action Handler 오류

## Fallback 처리

- 원본 Component 렌더링 중단
- errorCard 표시
- 재시도 가능 여부 설정
- requestId 기록
- 내부 상세 정보 비노출

# 오류 분류

## Response 오류

- responseParseError
- responseValidationError
- contractVersionError

## UI 오류

- componentNotFound
- propsValidationError
- renderError

## Action 오류

- actionNotFound
- invalidActionPayload
- invalidStateTransition
- duplicateAction

## API 오류

- chatRequestError
- inquiryRequestError
- encryptionError
- decryptionError
- timeoutError

# 로깅 및 관측

## 기록 항목

- requestId
- contractVersion
- component
- action type
- Validation 결과
- State Transition
- Action 실행 결과
- API 처리 시간
- Error Category

## 기록 제외 항목

- 문의 전문
- 이메일
- 전화번호
- 개인정보
- AES Key
- 복호화 Payload 전체
- System Prompt

# 접근성

## Button

- 의미 있는 Label 제공
- Disabled 상태 명확화
- Keyboard 조작 지원

## Dynamic Message

- 적절한 Live Region 적용
- 새 응답 도착 안내
- 오류 발생 안내

## Form

- Label과 Input 연결
- 오류 메시지 연결
- 필수값 표시
- Focus 이동 관리

# Frontend 완료 조건

## Component 안전성

- 미등록 Component가 렌더링되지 않는다.
- Backend 문자열을 Component 경로로 직접 사용하지 않는다.
- 모든 Component가 Registry를 통해 선택된다.

## Action 안전성

- 미등록 Action이 실행되지 않는다.
- 모든 Action이 Dispatcher를 통과한다.
- Action Payload가 실행 전에 검증된다.
- 동적 코드 실행을 사용하지 않는다.

## 상태 관리

- 모든 문의 상태 전이가 State Machine Guard를 통과한다.
- 전송 중 중복 요청이 발생하지 않는다.
- 오류 발생 시 Draft가 유지된다.
- 실제 API 성공 후에만 sent 상태로 변경된다.

## UI 책임 분리

- UI Component가 API를 직접 호출하지 않는다.
- UI Component가 Store를 직접 변경하지 않는다.
- UI Component는 Event만 상위로 전달한다.
- Renderer와 Dispatcher의 책임이 분리된다.

## 오류 처리

- 잘못된 응답은 errorCard로 대체된다.
- Contract Version 불일치를 안전하게 처리한다.
- 내부 오류 정보가 사용자에게 노출되지 않는다.