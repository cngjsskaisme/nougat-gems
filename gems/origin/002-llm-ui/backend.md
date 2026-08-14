# json-render 기반 Backend UI 생성 레이어 설계서

> 기준: json-render 공식 문서의 Catalog Prompt, Inline Mode, SpecStream, Mixed Stream Processing, Spec Validation 모델  
> 범위: LLM이 Catalog에 제한된 UI Spec을 생성하고 Frontend에 전달하는 계층

# 1. 설계 목적

본 설계의 목적은 사용자 요청과 대화 Context를 기반으로 LLM이 안전하고 예측 가능한 UI Spec을 생성하도록 하는 Backend UI 생성 레이어를 정의하는 것이다.

Backend UI 생성 레이어의 표준 흐름은 다음과 같다.

```text
Catalog
→ Catalog Prompt
→ LLM
→ Inline Mixed Stream
→ Text와 SpecStream 분리
→ Spec Compile
→ Catalog Validation
→ Text Part와 Spec Part 반환
```

LLM은 HTML, Vue Component 또는 실행 코드를 생성하지 않는다.

LLM은 Catalog가 허용한 Component, Props, Action만 사용하여 json-render Spec을 생성한다.

# 2. 적용 모드

# 2.1 Inline Mode

대화형 UI 생성에는 json-render의 Inline Mode를 사용한다.

Backend는 `catalog.prompt({ mode: "inline" })`을 사용하여 LLM System Prompt를 생성한다.

Inline Mode에서 LLM은 다음을 반환할 수 있다.

- 일반 텍스트
- 일반 텍스트와 JSONL Patch
- JSONL Patch 중심 응답

Backend는 텍스트와 JSONL Patch를 구분하여 순서가 유지된 Message Part로 변환한다.

# 2.2 Standalone Mode 제외

Standalone Mode는 전체 결과가 하나의 UI Spec인 생성 도구에 적합하다.

본 설계는 대화 안에서 UI를 생성하는 Backend 계층을 대상으로 하므로 Standalone Mode는 기본 범위에 포함하지 않는다.

# 3. Backend UI 생성 레이어의 책임

Backend UI 생성 레이어는 다음을 담당한다.

- Catalog 관리
- Catalog Prompt 생성
- 대화 Context와 UI 생성 규칙 결합
- LLM 호출
- Inline Mixed Stream 처리
- Text와 SpecStream 분리
- SpecStream Compile
- 최종 Spec 구조 검증
- Catalog 기반 Component·Props·Action 검증
- UI 생성 정책 검증
- Text Part와 Spec Part 구성
- SpecStream 오류 처리
- UI 생성 관련 관측성

Backend UI 생성 레이어는 다음을 담당하지 않는다.

- Vue Component 구현
- Frontend Rendering
- UI State의 실제 관리
- 사용자 Event 처리
- 업무 Workflow 실행
- 외부 시스템 호출

# 4. 주요 구성 요소

# 4.1 Catalog

Catalog는 AI가 생성할 수 있는 UI의 어휘를 정의한다.

Catalog는 다음 항목을 포함한다.

- Component 정의
- Component Props Schema
- Slot 정의
- Component Description
- Action 정의
- Action Parameter Schema
- Action Description
- Custom Function 정의

Catalog는 생성 제약과 검증 계약을 동시에 제공한다.

# 4.2 Catalog Prompt Generator

Catalog Prompt Generator는 Catalog를 LLM용 System Prompt로 변환한다.

Inline Mode에서는 다음을 사용한다.

```text
catalog.prompt({ mode: "inline" })
```

생성된 Prompt에는 다음이 포함된다.

- 사용 가능한 Component
- Component Props
- Slot
- 사용 가능한 Action
- Action Parameter
- SpecStream 작성 형식
- Inline Text 작성 규칙

# 4.3 UI Generation Rules

Catalog Prompt에는 UI 생성 목적에 맞는 Custom Rule을 추가한다.

Custom Rule은 다음을 정의할 수 있다.

- UI를 생성해야 하는 조건
- 텍스트만 반환해야 하는 조건
- 사용을 권장하는 Component
- 사용을 제한하는 Component
- 최대 UI 복잡도
- Layout 구성 원칙
- UI에 포함할 수 있는 정보 범위
- Action 노출 조건

Custom Rule은 UI 생성 방향을 유도하는 규칙이다.

Catalog와 Validation을 대신하지 않는다.

# 4.4 LLM Gateway

LLM Gateway는 UI 생성 레이어와 LLM Provider를 분리한다.

Gateway는 다음 공통 입력을 받는다.

- System Prompt
- Conversation
- Generation Mode
- 출력 제한
- Streaming 여부

Gateway는 Text Stream을 반환한다.

Inline Mode의 Text Stream에는 일반 텍스트와 JSONL Patch가 함께 포함될 수 있다.

# 4.5 Mixed Stream Processor

Mixed Stream Processor는 Inline Mode 출력에서 일반 텍스트와 SpecStream Patch를 구분한다.

공식 json-render Utility를 사용한다.

사용 가능한 공식 처리 방식은 다음과 같다.

- `pipeJsonRender`
- `createJsonRenderTransform`
- `createMixedStreamParser`

Mixed Stream Processor는 자체 정규식 기반 파서를 만들지 않고 공식 Parsing 규칙을 사용한다.

# 4.6 SpecStream Compiler

SpecStream Compiler는 JSONL Patch를 순서대로 적용하여 현재 Spec을 구성한다.

공식 처리 방식은 다음과 같다.

- `compileSpecStream`
- `createSpecStreamCompiler`

Compiler는 다음 상태를 관리한다.

- 현재 Spec
- 적용된 Patch
- 마지막 Patch
- Compile 오류
- 완료 여부

# 4.7 Spec Validator

Spec Validator는 Compile된 Spec이 Schema와 Catalog에 맞는지 확인한다.

공식 기능은 다음과 같다.

- `validateSpec()`
- `catalog.validate()`
- `catalog.zodSchema()`
- `catalog.jsonSchema()`

Spec Validator는 구조 검증과 Catalog 검증을 분리하여 수행한다.

# 4.8 Response Part Composer

Response Part Composer는 Inline Mode 결과를 Frontend가 렌더링할 Part로 구성한다.

논리 Part는 다음과 같다.

- Text Part
- Spec Part

Part의 순서는 LLM 출력 순서를 유지한다.

검증을 통과한 Spec만 Spec Part에 포함한다.

# 5. Catalog 설계

# 5.1 Catalog와 Schema

Schema는 Spec의 구조와 문법을 정의한다.

Catalog는 해당 구조 안에서 사용할 수 있는 Component, Action, Function의 목록을 정의한다.

```text
Schema
= Spec의 문법

Catalog
= 생성 가능한 UI의 어휘
```

Backend는 대상 Frontend Renderer와 호환되는 Schema로 Catalog를 정의한다.

# 5.2 Component 정의

Catalog Component는 다음 정보를 가진다.

- Props Schema
- 선택적 Slot
- Description

Props Schema는 AI가 생성할 수 있는 Property의 형식과 허용 값을 제한한다.

Description은 LLM이 Component의 목적과 적절한 사용 시점을 판단하도록 돕는다.

# 5.3 Action 정의

Catalog Action은 다음 정보를 가진다.

- Action 이름
- Parameter Schema
- Description

AI는 정의된 Action 이름과 Parameter만 Spec에 포함할 수 있다.

Action 실행 구현은 Catalog에 포함하지 않는다.

# 5.4 Function 정의

Catalog Function은 Validation 또는 Transformation에서 사용할 수 있는 이름 있는 Function을 선언한다.

LLM은 Catalog에 등록된 Function만 참조할 수 있다.

실제 Function 구현은 Runtime에서 제공한다.

# 5.5 Catalog 규모

Catalog는 UI 생성을 위해 필요한 Component와 Action으로 제한한다.

과도한 Catalog는 다음 문제를 만든다.

- Prompt 크기 증가
- Component 선택의 모호성
- 생성 결과 일관성 저하
- 검증 범위 확대
- Spec 복잡도 증가

Catalog는 제품의 UI Design System과 생성 목적을 반영해야 한다.

# 6. Prompt 구성

UI 생성용 System Prompt는 다음 순서로 구성한다.

```text
json-render Inline Mode 지침
→ Catalog Component 정의
→ Catalog Action 정의
→ Catalog Function 정의
→ UI 생성 Custom Rule
→ 대화 Context
```

Prompt에서 사용자 요청과 Catalog 규칙을 명확히 구분한다.

LLM이 Catalog 밖의 Component 또는 Action을 생성하지 않도록 Catalog Prompt를 생성의 기준으로 사용한다.

# 7. Inline Mixed Stream 처리

# 7.1 Text Line

JSON Patch로 해석되지 않는 내용은 Text Part로 전달한다.

# 7.2 Patch Line

유효한 JSONL Patch Line은 SpecStream으로 전달한다.

# 7.3 순서 보존

텍스트와 Patch가 번갈아 출력될 수 있으므로 Mixed Stream Processor는 Part 경계를 보존한다.

예를 들어 텍스트, Spec, 텍스트 순으로 생성되면 Response Part도 동일한 순서를 유지한다.

# 7.4 불완전 Chunk

Streaming Transport에서는 하나의 JSONL Line이 여러 Chunk로 나뉠 수 있다.

Parser는 Chunk가 아닌 완성된 Line 기준으로 Patch를 판정한다.

# 8. SpecStream 설계

# 8.1 Patch 형식

SpecStream은 RFC 6902 JSON Patch Operation을 사용한다.

지원 Operation은 다음과 같다.

- add
- remove
- replace
- move
- copy

# 8.2 Patch 적용 순서

Patch는 LLM이 출력한 순서대로 적용한다.

순서를 변경하거나 병렬 적용하지 않는다.

# 8.3 점진적 Spec

Patch가 적용될 때마다 현재 Spec을 갱신할 수 있다.

점진적 Spec은 Frontend Streaming Rendering에 전달하거나 Backend 내부 최종 Spec 생성에 사용할 수 있다.

# 8.4 최종 Spec

Stream이 정상 완료되면 마지막 현재 Spec을 최종 Spec으로 확정한다.

불완전 Line 또는 적용 실패 Patch가 남아 있으면 최종 성공으로 처리하지 않는다.

# 9. Spec Validation

# 9.1 구조 검증

구조 검증은 Spec이 선택된 Schema의 문법을 따르는지 확인한다.

검증 항목:

- Spec Object 형식
- Root 존재
- Elements 구조
- Element Type
- Props 구조
- Children 참조
- Event Binding 구조
- Dynamic Value 형식
- State 형식

# 9.2 Catalog 검증

Catalog 검증은 Spec이 허용된 UI 어휘만 사용하는지 확인한다.

검증 항목:

- Component 등록 여부
- Component Props Schema
- Slot 사용
- Action 등록 여부
- Action Parameter Schema
- Function 등록 여부

# 9.3 UI 생성 정책 검증

UI 생성 정책 검증은 Catalog Schema로 표현하기 어려운 생성 품질 규칙을 확인한다.

검증 항목:

- 최대 Element 수
- 최대 중첩 수준
- 최대 반복 수
- 허용 Layout 범위
- 동일 Component 과다 반복
- 불필요한 UI 생성
- Text Only가 적합한 요청에서 과도한 Spec 생성
- UI에 포함할 수 없는 정보

# 9.4 검증 실패

검증에 실패한 Spec은 Frontend로 전달하지 않는다.

처리 방식은 다음 중 하나다.

- 유효한 Text Part만 반환
- UI 생성 실패 Part 반환
- 제한된 재생성 시도
- 해당 응답 실패 처리

원본의 잘못된 Spec은 렌더링 대상으로 사용하지 않는다.

# 10. Response 설계

# 10.1 Streaming Response

Streaming Response에서는 Text Part와 Spec Patch Part를 순차적으로 전달한다.

Frontend는 Spec Patch를 Compile하면서 점진적으로 렌더링한다.

# 10.2 Final Spec Response

Non-streaming Response에서는 Backend가 SpecStream을 모두 Compile하고 검증한 뒤 최종 Spec Part를 반환한다.

# 10.3 Response Part 원칙

- Text와 Spec을 별도 Part로 표현한다.
- Part 순서를 유지한다.
- Spec Part에는 검증된 Spec만 포함한다.
- Text Only 응답을 허용한다.
- UI Only 응답을 허용한다.
- Text와 UI가 함께 있는 응답을 허용한다.

# 11. 오류 처리

UI 생성 레이어의 오류 유형은 다음과 같다.

- Catalog Prompt 생성 오류
- LLM 생성 오류
- Mixed Stream Parse 오류
- JSONL Line 오류
- Patch 적용 오류
- Spec Compile 오류
- 구조 검증 오류
- Catalog 검증 오류
- UI 생성 정책 오류

오류 발생 시 전체 서버 내부 정보를 반환하지 않고 UI 생성 결과의 성공 여부와 제한된 오류 분류만 제공한다.

# 12. 버전 관리

Backend UI 생성 레이어는 다음 버전을 관리한다.

- Catalog Version
- Schema Version
- SpecStream Format Version
- Response Part Contract Version
- UI Generation Rule Version

Frontend가 지원하는 Catalog 및 Schema Version과 일치하는 Spec만 전달한다.

Catalog 변경 시 Prompt, Validation, Frontend Registry에 미치는 영향을 함께 검토한다.

# 13. 관측성

UI 생성 레이어는 다음 정보를 기록한다.

- Generation Mode
- Catalog Version
- Schema Version
- LLM Provider와 Model
- Text Part 수
- Patch 수
- 최종 Element 수
- Spec Compile 결과
- Catalog Validation 결과
- UI 생성 정책 결과
- 생성 시간
- Streaming 완료 또는 중단
- 오류 유형

관측 정보는 UI 생성 과정의 품질과 안정성을 평가하는 데 사용한다.

# 14. 성능 원칙

- Catalog Prompt 크기를 제한한다.
- 대화 Context 크기를 제한한다.
- 최대 Patch 수를 제한한다.
- 최대 Spec 크기를 제한한다.
- 최대 Element 수를 제한한다.
- 최대 State 크기를 제한한다.
- Text Only 응답을 불필요하게 Spec으로 변환하지 않는다.
- Validation 실패 시 무제한 재생성을 하지 않는다.
- Streaming 중 불완전 Spec의 검증 비용을 관리한다.

# 15. 테스트 전략

# 15.1 Catalog Test

- Catalog Prompt 생성
- Component Props Schema
- Action Parameter Schema
- Function 정의
- Catalog Version

# 15.2 Generation Test

- Text Only
- Spec Only
- Text와 Spec 혼합
- 여러 Text Block과 Spec Part
- Catalog Component 선택 정확성
- Action 생성 정확성

# 15.3 Streaming Test

- 정상 JSONL Patch
- 여러 Chunk로 분할된 Patch
- 불완전 Line
- 잘못된 Patch Operation
- Patch 순서 오류
- Stream 중단
- 최종 Spec 확정

# 15.4 Validation Test

- 정상 Spec
- 미등록 Component
- 잘못된 Props
- 미등록 Action
- 잘못된 Action Parameter
- 미등록 Function
- 잘못된 Children 참조
- UI 복잡도 제한 초과

# 16. 완료 조건

Backend UI 생성 레이어는 다음 조건을 만족해야 한다.

- `catalog.prompt({ mode: "inline" })`을 기준으로 LLM Prompt를 생성한다.
- Inline Output에서 Text와 JSONL Patch를 공식 Utility로 분리한다.
- SpecStream을 최종 또는 점진적 Spec으로 Compile한다.
- 최종 Spec을 구조적으로 검증한다.
- 최종 Spec을 `catalog.validate()`로 검증한다.
- Catalog 밖의 Component, Action, Function을 제거하거나 거부한다.
- 검증된 Spec만 Frontend에 전달한다.
- Text Only, Spec Only, Text와 Spec 혼합 응답을 지원한다.
- Streaming과 Final Spec Response를 동일한 Spec 모델로 처리한다.
- Frontend Registry와 Catalog Version의 호환성을 유지한다.

# 17. 공식 문서 기준

- Introduction: https://json-render.dev/docs
- Catalog: https://json-render.dev/docs/catalog
- Generation Modes: https://json-render.dev/docs/generation-modes
- Specs: https://json-render.dev/docs/specs
- Streaming: https://json-render.dev/docs/streaming
- Registry: https://json-render.dev/docs/registry
- Validation: https://json-render.dev/docs/validation
- Core API: https://json-render.dev/docs/api/core
- AI SDK Integration: https://json-render.dev/docs/ai-sdk
