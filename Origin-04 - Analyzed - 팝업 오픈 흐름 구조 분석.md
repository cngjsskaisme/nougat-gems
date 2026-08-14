# 팝업 오픈 흐름 구조 분석

> 분석 대상: nougat-snippet 내 설계 문서에서 발견된 팝업/오픈 흐름
> 기준 문서:
> - `이력서 채팅 모듈.md`
> - `자체개발 - LLM UI FE.md`
> - `자체개발 - LLM UI BE.md`
> - `Mixed-01-02 - NDJSON_json-render_접합_FE_프로그램_설계서.md`
> - `Origin-02 - 잠재표준 - LLM UI FE.md`

---

## 1. 핵심 결론

현재 설계 문서들에는 단일한 "팝업" 개념보다는 **4개의 서로 다른 오픈/표시 흐름**이 존재한다.

| 흐름 | 트리거 | 상태/표시 제어 주체 | 대표 사례 |
|------|--------|---------------------|-----------|
| A. 채팅 위젯 오픈 | 사용자 클릭 / 외부 진입 | `useChatState` (`desktopOpen` / `mobileOpen`) | 이력서 채팅 모듈 |
| B. 등록 링크 오픈 | UI Action (`openRegisteredLink`) | `Action Dispatcher` → `Link Registry` | LLM UI 문의 카드 |
| C. 상품 상세 오픈 | UI Action (`openProductDetail`) | `Action Dispatcher` → Navigation Handler | NDJSON-json-render 상품 버튼 |
| D. 조건부 UI 노출 | Spec 상태 변화 | `VisibilityProvider` (`visible`) | json-render 기반 생성 UI |

이 4가지 흐름은 공통적으로 **"UI Component는 직접 열지 않고, 상태 또는 Dispatcher를 거친다"**는 원칙을 따른다.

---

## 2. 공통 아키텍처

```text
사용자 이벤트 (클릭, 진입, 상태 변화)
        ↓
[경우에 따라] UI Component는 Event만 Emit
        ↓
State Layer 또는 Action Dispatcher
        ↓
검증 (Registry, Allowlist, State Machine Guard)
        ↓
실제 오픈/렌더링 수행 (Router, window.open, 조건부 렌더링)
```

### 공통 원칙

- UI Component는 직접 API 호출, 직접 State 변경, 직접 Action Handler 선택을 하지 않는다.
- 오픈 대상은 사전에 등록된 값(ID, URL, Component)만 사용한다.
- 임의 문자열을 함수명, Import 경로, URL로 사용하지 않는다.
- 비동기 Action의 Loading과 Error는 UI Part 범위에서 관리한다.

---

## 3. 흐름별 상세 구조

### A. 채팅 위젯 오픈/닫힘

`이력서 채팅 모듈.md`에 정의된 구조다.

#### 상태

```text
useChatState
├── messages
├── loading
├── desktopOpen   ← 데스크톱 채팅 위젯/팝업 노출 상태
├── mobileOpen    ← 모바일 채팅 위젯/팝업 노출 상태
├── draft
└── inquiryState
```

#### 흐름

```text
[데스크톱]
사용자 클릭 (FAB / 토글 버튼)
        ↓
useChatState.desktopOpen = !desktopOpen
        ↓
DesktopChat Component 조건부 렌더링
        ↓
채팅 팝업/위젯 노출

[모바일]
사용자 클릭
        ↓
useChatState.mobileOpen = !mobileOpen
        ↓
MobileChat Component 조건부 렌더링
        ↓
전면/바텀 시트 형태 채팅 UI 노출
```

#### 특징

- 별도의 `Dialog`/`Modal` 컴포넌트는 언급되지 않는다.
- `desktopOpen` / `mobileOpen`은 메모리 내 `Application State`로 관리된다.
- 새로고침 시 초기화된다.
- 뷰포트별로 별도 상태를 두어 반응형 오픈 정책을 적용할 수 있다.

---

### B. 등록 링크 오픈

`openRegisteredLink` Action을 중심으로 한 흐름이다.

#### 등장 위치

- `자체개발 - LLM UI FE.md` / `자체개발 - LLM UI BE.md`
- `Action Registry`에 등록된 Action 중 하나

#### 흐름

```text
사용자 클릭
        ↓
UI Component가 action Event Emit
        ↓
Response Renderer가 수신 → Action Dispatcher로 전달
        ↓
Action Registry에서 openRegisteredLink Handler 조회
        ↓
Payload 검증
        ↓
Link Registry에서 Link ID → 실제 URL 조회
        ↓
보안 검증
        ↓
window.open 또는 route 이동
```

#### 보안 검증 항목

- 등록된 Link ID인지
- 허용 Protocol인지
- 허용 Host인지
- 상대경로인지 절대경로인지
- 새 창 정책
- `noopener` 및 `noreferrer` 적용 여부

#### 차단 항목

- `javascript:` scheme
- `data:` scheme
- 임의 iframe
- 등록되지 않은 Domain
- LLM이 생성한 검증되지 않은 URL

#### 핵심 원칙

> LLM 또는 Backend는 URL 대신 Link ID를 전달한다. Frontend는 Link Registry에서 실제 URL을 조회한다.

---

### C. 상품 상세 오픈

`openProductDetail` Action을 중심으로 한 흐름이다.

#### 등장 위치

- `Mixed-01-02 - NDJSON_json-render_접합_FE_프로그램_설계서.md` #18 Action 연결

#### Spec 예시

```json
{
  "type": "Button",
  "props": {
    "label": "상세 보기"
  },
  "on": {
    "click": [
      {
        "action": "openProductDetail",
        "params": {
          "productId": "p-01"
        }
      }
    ]
  }
}
```

#### 흐름

```text
사용자가 Button 클릭
        ↓
content.ui.patch 기반 UI Part에서 click Event 발생
        ↓
Renderer가 Action Event를 Action Dispatcher로 전달
        ↓
Action Registry에서 openProductDetail Handler 조회
        ↓
Payload 검증 (productId 존재 여부 등)
        ↓
Navigation URL Allowlist 내 페이지 이동 또는 팝업 오픈
```

#### Frontend 원칙

- Catalog/Registry에 등록된 Action만 실행
- Parameter Schema 재검증
- 임의 Function String 실행 금지
- Navigation URL Allowlist
- 비동기 Action의 Loading과 Error를 UI Part 범위에서 관리
- 서버 권한이 필요한 Action은 서버에서 다시 권한 검증
- Visibility는 권한 검증 수단으로 사용하지 않음

---

### D. 조건부 UI 노출 (Visibility)

`json-render` Spec의 `visible` 속성을 활용한 간접적 팝업/표시 흐름이다.

#### 등장 위치

- `Origin-02 - 잠재표준 - LLM UI FE.md` #5.2 VisibilityProvider
- `Mixed-01-02 - NDJSON_json-render_접합_FE_프로그램_설계서.md` UI Patch 처리 흐름

#### Spec 예시

```json
{
  "type": "Dialog",
  "props": { "title": "확인" },
  "visible": "$state.showConfirmDialog",
  "children": [...]
}
```

#### 흐름

```text
content.ui.patch
        ↓
Payload Schema Validation
        ↓
Patch Guard
        ↓
Part Compiler 적용
        ↓
workingSpec 갱신
        ↓
Boundary Validation
        ↓
renderSpec으로 Promote
        ↓
Vue Renderer
        ↓
VisibilityProvider가 $state.showConfirmDialog 평가
        ↓
조건 충족 시 Dialog/Panel 표시
```

#### VisibilityProvider 책임

- Spec에 선언된 조건에 따라 Element 표시 여부 결정
- State 경로를 참조
- 조건부 Component 표시
- 입력값에 따른 추가 UI 표시
- 선택 상태에 따른 Section 변경
- UI 내부 Loading 또는 Error 표현

#### 주의

> Visibility는 화면 표현만 담당한다. 업무 권한이나 서버 실행 가능 여부를 판단하지 않는다.

---

## 4. 통합 팝업 오픈 구조

```text
                          사용자 상호작용
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   FAB/토글 클릭            UI Action 클릭          상태 변화
        │                      │                      │
        ▼                      ▼                      ▼
  useChatState          Action Dispatcher        VisibilityProvider
 desktopOpen /          (openRegisteredLink,      ($state 기반)
 mobileOpen             openProductDetail)
        │                      │                      │
        ▼                      ▼                      ▼
 조건부 렌더링         Registry / Link Registry   조건부 렌더링
        │                      │                      │
        ▼                      ▼                      ▼
DesktopChat /         window.open / router.push   Dialog / Panel
MobileChat                                 or
                                      Product Detail Page
```

---

## 5. 팝업 유형별 적합한 패턴 선택

| 팝업 유형 | 추천 패턴 | 이유 |
|-----------|-----------|------|
| 채팅 위젯/플로팅 UI | A. 채팅 위젯 오픈 | 뷰포트별 글로벌 상태가 필요하고, 메시지 스트림과 생명주기를 공유 |
| 외부 사이트/등록 링크 | B. 등록 링크 오픈 | 보안 검증과 URL 추상화가 필요 |
| 내부 상세 페이지/모달 | C. 상품 상세 오픈 | Action 기반으로 파라미터 검증과 네비게이션 정책 적용 |
| 생성형 UI 내부 조건부 표시 | D. Visibility | Spec 기반으로 동적인 UI 노출/숨김 처리 |

---

## 6. 보안 체크리스트

- [ ] UI Component가 직접 `window.open`을 호출하지 않는다.
- [ ] Action은 등록된 Handler를 통해서만 실행된다.
- [ ] 외부 링크는 등록된 Link ID와 Allowlist를 통과한다.
- [ ] LLM이 생성한 URL은 검증되지 않은 한 사용되지 않는다.
- [ ] `noopener`, `noreferrer`가 새 창/탭 오픈에 적용된다.
- [ ] `javascript:`, `data:` scheme은 차단된다.
- [ ] Visibility는 권한 검증이 아닌 표시 제어로만 사용된다.
- [ ] 서버 권한이 필요한 Action은 서버에서 재검증된다.

---

## 7. 열려 있는 질문

1. **통합 팝업 레이어 필요 여부**: 4개 패턴을 하나의 `PopupManager`/`DialogStore`로 통합할 것인가, 현재처럼 분리 유지할 것인가?
2. **A11y**: 각 오픈 흐름에 대한 포커스 트랩, ESC 닫기, `aria-modal` 처리 방식은?
3. **뒤로 가기 처리**: 모바일 채팅(`mobileOpen`)과 Dialog(`visible`)이 브라우저 뒤로 가기와 어떻게 상호작용하는가?
4. **애니메이션**: 열기/닫기 트랜지션은 각 패턴에서 어떻게 처리되는가?
5. **z-index/포탈**: 여러 팝업이 동시에 열릴 경우 레이어 관리 정책은?

---

## 8. 참고

- `이력서 채팅 모듈.md`: Chat State Layer, `desktopOpen` / `mobileOpen`
- `자체개발 - LLM UI FE.md`: Component Registry, Action Registry, Action Dispatcher, `openRegisteredLink`
- `자체개발 - LLM UI BE.md`: Backend Action 정의, `openRegisteredLink`
- `Mixed-01-02 - NDJSON_json-render_접합_FE_프로그램_설계서.md`: Action 연결, `openProductDetail`
- `Origin-02 - 잠재표준 - LLM UI FE.md`: VisibilityProvider, ActionProvider
