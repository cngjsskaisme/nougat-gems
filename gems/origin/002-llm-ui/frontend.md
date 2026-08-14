# json-render 기반 Frontend UI 생성 레이어 설계서

> 기준: json-render 공식 문서의 Catalog, Vue Renderer, Inline Mode, SpecStream, State, Action, Visibility, Validation 모델  
> 범위: AI가 생성한 UI Spec을 Vue UI로 구성하고 렌더링하는 계층 

# 1. 설계 목적

본 설계의 목적은 AI가 생성한 UI를 Vue 애플리케이션에서 안전하고 일관된 방식으로 렌더링하는 Frontend UI 생성 레이어를 정의하는 것이다.

기존의 응답 타입별 분기 구조를 다음과 같은 json-render 공식 흐름으로 전환한다.

```text
AI가 생성한 Spec
→ Catalog와의 호환성 확인
→ Vue Registry 조회
→ Provider를 통한 상태·동작 해석
→ Renderer
→ Vue Component Tree
```

AI는 Vue 코드나 HTML을 생성하지 않는다. AI는 Catalog에 정의된 Component, Props, Action만 사용하여 JSON Spec을 생성한다.

# 2. 적용 모드

# 2.1 Inline Mode

대화형 UI에서는 json-render의 Inline Mode를 사용한다.

Inline Mode에서는 하나의 AI 응답이 다음 형태를 가질 수 있다.

- 텍스트만 포함
- UI Spec만 포함
- 텍스트와 UI Spec을 함께 포함

Frontend는 Message 안에서 Text Part와 Spec Part를 구분하여 렌더링한다.

# 2.2 Standalone Mode 제외

Standalone Mode는 전체 결과가 하나의 생성형 UI인 화면 생성기, 대시보드 생성기, 폼 빌더 등에 적합하다.

본 설계는 대화 안에 생성형 UI를 삽입하는 구조이므로 Standalone Mode를 기본 범위에 포함하지 않는다.

# 3. Frontend UI 생성 레이어의 책임

Frontend UI 생성 레이어는 다음을 담당한다.

- 수신된 Message Part 구분
- Spec Part 추출
- Spec의 기본 구조 확인
- Catalog와 Vue Registry 연결
- Vue Component Tree 렌더링
- Generated UI State 제공
- State Binding 해석
- Visibility 조건 평가
- Validation 실행
- Action Event와 Handler 연결
- SpecStream의 점진적 반영
- 알 수 없는 Component에 대한 Fallback 처리

Frontend UI 생성 레이어는 다음을 담당하지 않는다.

- LLM 호출
- Catalog Prompt 생성
- AI 출력 생성
- 업무 Workflow 판정
- 외부 시스템 실행
- 서버 결과 판정

# 4. 주요 구성 요소

# 4.1 Message Part Resolver

Message Part Resolver는 Inline Mode 응답에서 텍스트와 UI Spec을 구분한다.

논리적인 Message Part는 다음과 같다.

- Text Part
- Spec Part

Text Part는 기존 텍스트 또는 Markdown Renderer로 전달한다.

Spec Part는 json-render Vue Renderer로 전달한다.

Message Part Resolver는 Part의 순서를 유지해야 한다. 텍스트와 UI가 교차하여 생성된 경우에도 원래 생성 순서대로 표시한다.

# 4.2 Spec Boundary Validator

Spec Boundary Validator는 Renderer 진입 전에 Spec이 처리 가능한 형식인지 확인한다.

검증 범위는 다음과 같다.

- Spec 존재 여부
- Spec Object 형식
- Root 식별자 존재 여부
- Elements 구조 존재 여부
- State 형식
- Element 참조 형식
- Catalog Version 호환 여부
- 최대 Element 수
- 최대 State 크기

Catalog의 Component별 상세 검증은 Backend에서 수행하더라도 Frontend 경계에서 기본 검증을 다시 수행한다.

# 4.3 Vue Registry

Vue Registry는 Catalog에 정의된 Component 이름을 실제 Vue Component 구현과 연결한다.

공식 `@json-render/vue`의 `defineRegistry`를 사용한다.

Registry는 다음을 제공한다.

- Catalog 기반 타입 안전성
- Component 구현 연결
- Action Handler 연결
- Component Event 전달
- Props 전달
- Children 전달
- Loading 상태 전달
- Binding 경로 전달

Registry는 Component 선택을 위한 유일한 진입점이다.

Spec에 포함된 문자열을 Vue Component 이름이나 Import 경로로 직접 해석하지 않는다.

# 4.4 Renderer

공식 `Renderer`는 검증된 Spec과 Registry를 입력받아 Vue Component Tree를 생성한다.

Renderer는 다음 순서로 동작한다.

```text
Spec Root 확인
→ Element 조회
→ Component Type 조회
→ Registry Component 선택
→ Props Dynamic Value 해석
→ Children 재귀 렌더링
→ Event Binding 연결
```

Renderer는 미등록 Component를 발견하면 임의로 추정하지 않고 Fallback Component를 사용한다.

# 5. Provider 설계

# 5.1 StateProvider

StateProvider는 Generated UI에서 사용하는 State Model을 제공한다.

StateProvider는 다음 두 운영 방식을 지원한다.

## Uncontrolled Mode

json-render 내부 Store가 UI State를 관리한다.

적합한 상태:

- UI 내부 입력값
- 일시적인 선택 상태
- Toggle 상태
- 조건부 표시 값
- 생성된 UI 내부에서만 사용하는 데이터

## Controlled Mode

외부 StateStore를 StateProvider에 주입한다.

적합한 상황:

- 생성형 UI의 상태를 애플리케이션 Store와 공유해야 하는 경우
- UI 상태를 다른 화면에서도 참조해야 하는 경우
- 동일 Spec을 재사용하거나 복원해야 하는 경우

UI 생성 레이어의 기본 원칙은 동일 상태의 소유자를 하나로 유지하는 것이다.

# 5.2 VisibilityProvider

VisibilityProvider는 Spec에 선언된 조건에 따라 Element의 표시 여부를 결정한다.

Visibility 조건은 State 경로를 참조한다.

주요 역할:

- 조건부 Component 표시
- 입력값에 따른 추가 UI 표시
- 선택 상태에 따른 Section 변경
- UI 내부 Loading 또는 Error 표현

Visibility는 화면 표현만 담당한다. 업무 권한이나 서버 실행 가능 여부를 판단하지 않는다.

# 5.3 ValidationProvider

ValidationProvider는 Generated UI의 입력 검증을 담당한다.

공식 Built-in Validator와 Catalog에 등록된 Custom Validation Function을 사용할 수 있다.

검증 범위:

- 필수값
- 이메일 형식
- 최소·최대 길이
- 숫자 범위
- 정규식
- 다른 Field와의 비교
- 조건부 필수값

검증 시점은 다음 중 하나로 구성한다.

- change
- blur
- submit

Validation 결과는 Generated UI State 및 Component 표현에 반영한다.

# 5.4 ActionProvider

ActionProvider는 Spec에 선언된 Action 이름을 Frontend Handler와 연결한다.

ActionProvider의 역할:

- Action 이름 조회
- Parameter 전달
- 비동기 Action 실행
- Action 실행 중 Loading 상태 제공
- Success 또는 Error 후속 Action 연결
- Navigation Handler 연결

AI는 Action 이름과 Parameter만 선언한다. 실제 Handler 구현은 Frontend가 제공한다.

# 6. Component 설계 원칙

# 6.1 Catalog 기반 Component

모든 생성 가능 Component는 Catalog에 먼저 정의되어야 한다.

각 Component 정의는 다음 정보를 가진다.

- Component 이름
- Props Schema
- 선택적 Slot
- AI가 Component 용도를 이해하기 위한 Description

Frontend는 Catalog 정의와 일치하는 Vue Component 구현을 Registry에 등록한다.

# 6.2 Component 책임

Registry Component는 다음만 담당한다.

- Props 표현
- Children 표현
- 사용자 Event 발생
- Binding된 값 표시 및 변경
- Loading 상태 표시
- Validation 결과 표시

Registry Component는 다른 Component를 선택하거나 Spec을 직접 변경하지 않는다.

# 6.3 Container Component

Children을 포함하는 Component는 Catalog에 Slot을 명시한다.

Renderer는 Spec의 Children 참조를 실제 Vue Child Tree로 변환하여 전달한다.

# 7. Dynamic Value와 Data Binding

json-render의 Dynamic Value 모델을 사용하여 Props와 State를 연결한다.

지원 개념은 다음과 같다.

- `$state`: State 값 조회
- `$bindState`: State 값 양방향 Binding
- `$item`: 반복 Scope의 현재 Item 조회
- `$bindItem`: 반복 Item 값 Binding
- `$index`: 반복 Index 조회
- `$cond`: 조건에 따른 값 선택
- `$template`: State 값을 포함한 문자열 생성
- `$computed`: Catalog Function 기반 계산

Frontend는 Dynamic Value를 임의 표현식으로 평가하지 않는다.

json-render Runtime이 정의한 Directive와 Resolver만 사용한다.

# 8. Spec 구조

UI Spec은 선택된 Schema에 따라 구성된다.

Vue Renderer에서 사용하는 기본 Flat Tree 구조는 다음 개념을 가진다.

- root
- elements
- state

# 8.1 root

렌더링을 시작할 최상위 Element의 식별자다.

# 8.2 elements

Element ID를 Key로 하는 Element Map이다.

각 Element는 다음 정보를 포함할 수 있다.

- type
- props
- children
- on
- visible
- repeat

# 8.3 state

Generated UI가 사용하는 초기 State Model이다.

# 9. Action Binding

Element의 Event는 하나 이상의 Action과 연결될 수 있다.

처리 흐름은 다음과 같다.

```text
사용자 Event
→ Registry Component emit
→ Element Event Binding 조회
→ ActionProvider
→ Action Handler 실행
→ UI State 또는 렌더링 결과 반영
```

여러 Action이 연결된 경우 Spec에 선언된 순서를 유지한다.

State 변경 목적의 Built-in Action과 애플리케이션이 정의한 Custom Action을 구분한다.

# 10. Streaming 설계

# 10.1 SpecStream

json-render는 JSONL 기반 SpecStream을 사용한다.

각 Line은 RFC 6902 JSON Patch Operation이다.

Patch가 순차적으로 적용되며 현재 Spec이 점진적으로 완성된다.

# 10.2 Vue Streaming Runtime

Frontend는 `@json-render/vue`의 Streaming Composable 또는 `@json-render/core`의 SpecStream Compiler를 사용한다.

Streaming 상태는 다음을 포함한다.

- 현재 Spec
- Streaming 진행 여부
- 마지막 Patch
- 오류
- Abort 상태
- 완료 상태

# 10.3 점진적 Rendering

Patch 적용 후 현재 Spec이 유효한 범위에서 Renderer를 갱신한다.

아직 생성되지 않은 Element로 인한 일시적인 불완전 상태를 처리할 수 있어야 한다.

Streaming 완료 시 최종 Spec을 확정한다.

# 11. Fallback 처리

다음 경우 Fallback UI를 사용한다.

- Spec 형식 오류
- Root 누락
- 미등록 Component
- 잘못된 Element 참조
- Dynamic Value 해석 실패
- Action Handler 부재
- Streaming Patch 적용 실패
- Renderer 오류

Fallback은 전체 Chat UI를 중단하지 않고 해당 Spec Part 범위에서만 표시한다.

# 12. 버전 관리

Frontend UI 생성 레이어는 다음 버전을 확인한다.

- Catalog Version
- Spec Schema Version
- Message Part Contract Version
- Registry Version

지원하지 않는 Version의 Spec은 렌더링하지 않는다.

Catalog 변경 시 Registry Component와 Action Handler의 완전성을 함께 검증한다.

# 13. 관측성

UI 생성 레이어는 다음 이벤트를 기록한다.

- Spec 수신
- Spec Validation 성공·실패
- Registry Component 조회 실패
- Renderer 시작·완료·실패
- Action 실행 시작·완료·실패
- Streaming 시작·완료·중단
- Fallback 발생
- Version 불일치

관측 데이터는 Spec 구조와 처리 결과 중심으로 기록하며 사용자 입력 원문은 기본 기록 대상에서 제외한다.

# 14. 성능 원칙

- Text Only 응답에는 Renderer를 생성하지 않는다.
- Spec을 Message 단위로 관리한다.
- 동일 Spec을 불필요하게 재Compile하지 않는다.
- Patch 적용 시 변경된 Element 중심으로 갱신한다.
- 최대 Element 수와 반복 수를 제한한다.
- Registry Component는 안정적인 Key를 사용한다.
- Catalog 규모를 UI 목적에 맞게 제한한다.

# 15. 접근성 원칙

접근성은 AI가 아닌 Registry Component가 보장한다.

Registry Component는 다음을 기본 제공해야 한다.

- 의미 있는 Label
- Keyboard 조작
- Focus 처리
- Form Error 연결
- Loading 상태 알림
- Dynamic Content 변경 안내
- 적절한 Semantic Element

# 16. 완료 조건

Frontend UI 생성 레이어는 다음 조건을 만족해야 한다.

- Inline Mode의 Text Part와 Spec Part를 구분할 수 있다.
- `@json-render/vue`의 Registry와 Renderer를 사용한다.
- StateProvider, VisibilityProvider, ValidationProvider, ActionProvider를 목적에 맞게 구성한다.
- Catalog에 없는 Component는 렌더링하지 않는다.
- Catalog에 없는 Action은 실행하지 않는다.
- Dynamic Value와 Data Binding을 json-render Runtime으로 해석한다.
- SpecStream을 점진적 Spec으로 구성할 수 있다.
- Rendering 오류를 Spec Part 단위 Fallback으로 처리한다.
- Catalog와 Registry Version의 호환성을 확인한다.

# 17. 공식 문서 기준

- Introduction: https://json-render.dev/docs
- Catalog: https://json-render.dev/docs/catalog
- Generation Modes: https://json-render.dev/docs/generation-modes
- Specs: https://json-render.dev/docs/specs
- Streaming: https://json-render.dev/docs/streaming
- Registry: https://json-render.dev/docs/registry
- Validation: https://json-render.dev/docs/validation
- Vue API: https://json-render.dev/docs/api/vue
