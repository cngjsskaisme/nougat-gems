# URL 기반 UI Action / Overlay Navigation 프로그램 설계서

## 1. 문서 목적

본 문서는 Vue 기반 SPA에서 화면, 팝업, 오버레이, 다이얼로그, 단계형 UI, 서브컴포넌트 호출 등 사용자가 인지할 수 있는 UI 상태를 URL Query String과 연결하여 추적·복원 가능하도록 만들기 위한 프로그램 설계를 정의한다.

기존 구조에서는 URL이 팝업의 종류와 대상을 나타내고, 팝업 내부의 입력값·로딩·오류 등의 상태는 로컬 세션 상태로 분리하는 방향을 사용한다.

본 설계에서는 이를 확장하여 다음을 목표로 한다.

* 모든 의미 있는 UI 상태를 일관된 Action 모델로 표현
* Query String의 Key를 `action`, `payload`, `meta`로 일원화
* 팝업, 페이지, 오버레이, 다이얼로그, 단계형 화면을 하나의 상태 모델로 관리
* 브라우저 뒤로가기/앞으로가기와 UI 상태를 연동
* 직접 URL 접근 및 새로고침 시 동일한 화면 복원
* 민감정보의 URL 노출 방지
* QA 및 E2E 테스트에서 특정 UI 상태로 직접 진입 가능
* 컴포넌트와 Vue Router의 직접 결합 최소화
* 팝업별 서로 다른 History 동작 지원
* Ephemeral UI가 이미 소비된 이후 Forward History로 다시 살아나는 현상 방지

---

# 2. 핵심 설계 원칙

## 2.1 URL은 UI 상태의 Source of Truth다

사용자가 현재 어떤 의미 있는 화면을 보고 있는지는 URL로 표현한다.

예:

```text
/
```

기본 화면

```text
/?action=account.view&payload=...
```

계좌 오버레이

```text
/?action=guestbook.create.message&payload=...
```

방명록 작성의 메시지 입력 단계

```text
/?action=guestbook.delete.auth&payload=...
```

방명록 삭제 인증 단계

기존 설계에서도 `route.query`를 상태 기준으로 삼아 직접 URL 접근, 새로고침, 브라우저 앞/뒤 이동에도 동일한 데이터 준비 흐름을 갖도록 하는 것을 권장하고 있다.

단, URL에는 모든 상태를 넣지 않는다.

URL에는 다음 조건을 만족하는 상태만 포함한다.

* 사용자가 인지할 수 있다.
* 새로고침 후 복원 가치가 있다.
* 브라우저 History와 연결할 가치가 있다.
* 특정 URL을 전달했을 때 같은 UI를 재현할 가치가 있다.

---

## 2.2 Query String의 Key는 3개로 고정한다

모든 UI 상태는 다음 형식을 사용한다.

```text
?action=<ACTION>
&payload=<PAYLOAD>
&meta=<META>
```

타 기능이 별도의 Query Key를 임의로 추가하는 것을 원칙적으로 금지한다.

예를 들어 다음과 같은 구조는 사용하지 않는다.

```text
?overlay=guestbook
&mode=delete
&target=123
&step=password
```

대신 다음과 같이 표현한다.

```text
?action=guestbook.delete.auth
&payload=<encoded-or-encrypted-value>
&meta=<encoded-value>
```

이를 통해 Query String의 인터페이스를 애플리케이션 전체에서 일관되게 유지한다.

---

# 3. Query 모델

## 3.1 QueryState

```ts
interface QueryState {
  action?: string
  payload?: string
  meta?: string
}
```

각 값의 책임은 다음과 같다.

### action

현재 UI 상태의 Identity를 나타낸다.

예:

```text
home

account.view
contact.view
chat.open

guestbook.view

guestbook.create.message
guestbook.create.name
guestbook.create.password

guestbook.modify.auth
guestbook.modify.edit

guestbook.delete.auth
guestbook.delete.confirm

directions.map.view
photo.preview
```

`action`에는 구현 컴포넌트 이름을 사용하지 않는다.

다음과 같은 표현은 피한다.

```text
AccountOverlay
GuestbookPasswordComponent
openModal
clickDeleteButton
```

대신 사용자 및 업무 관점의 의미를 사용한다.

```text
account.view
guestbook.delete.auth
```

즉:

```text
action
= 현재 사용자가 어떤 의미 있는 UI 상태에 있는가
```

를 나타낸다.

---

## 3.2 payload

`payload`는 해당 Action이 동작하는 데 필요한 데이터를 표현한다.

논리적 데이터 예:

```json
{
  "target": "groom"
}
```

```json
{
  "messageId": "123"
}
```

```json
{
  "photoId": "photo_3"
}
```

URL에서는 이를 직접 여러 Query Key로 분산시키지 않고 하나의 payload로 직렬화한다.

```text
?action=account.view
&payload=<serialized-value>
```

민감정보가 포함된 경우 암호화한다.

```text
?action=contact.view
&payload=<encrypted-base64url>
```

---

## 3.3 meta

`meta`는 Action의 업무 데이터와 분리된 부가 정보를 표현한다.

예:

```json
{
  "v": 1,
  "from": "gift",
  "presentation": "overlay"
}
```

단 다음처럼 현재 상태의 Identity를 표현하는 값을 meta에 넣는 것은 지양한다.

```json
{
  "mode": "delete",
  "step": "password"
}
```

이 경우 상태 자체를 다음과 같이 action으로 표현한다.

```text
guestbook.delete.auth
```

즉 기본 원칙은 다음과 같다.

```text
action
= 어디에 있는가

payload
= 무엇을 대상으로 하는가

meta
= 해당 상태에 부가적으로 필요한 문맥은 무엇인가
```

---

# 4. URL 상태에 포함하지 않는 값

다음 값들은 Query String에 저장하지 않는다.

```text
draft.message
draft.name
draft.password

loading
submitting
error

focusedField
composing
longPressTriggered

scrollLock
focusTrap
transitioning
zIndex
```

기존 설계에서도 방명록의 draft, submitting, error 등은 route가 아닌 기능별 세션 상태로 분리하도록 하고 있다.

---

# 5. 전체 상태 분류

애플리케이션의 상태는 네 계층으로 분리한다.

## 5.1 Navigable UI State

Query String에 저장한다.

```text
action
payload
meta
```

특징:

* 새로고침 복원 가능
* 딥링크 가능
* 브라우저 History와 연결 가능
* QA 재현 가능

---

## 5.2 Domain Session State

기능별 composable, ref/reactive 또는 Pinia 등에 저장한다.

예:

```text
draft.message
draft.name
draft.password

copied
submitting
error
```

이 상태는 URL 이동 자체와는 직접 연결하지 않는다.

---

## 5.3 UI Infrastructure State

공통 Overlay 인프라에서 관리한다.

예:

```text
scrollLock
focusTrap
previousFocus
ESC handler
backdrop
transition
zIndex
```

기존 설계에서도 이러한 공통 UI 제어를 `ModalShell` 또는 `OverlayHost`에서 관리하도록 하고 있다.

---

## 5.4 Event / Analytics State

사용자가 무엇을 했는지는 URL이 아니라 별도의 이벤트로 기록한다.

예:

```text
account.open
account.copy

contact.call
contact.sms

guestbook.create.start
guestbook.create.submit
guestbook.create.success
guestbook.create.failure
```

URL과 이벤트의 차이는 다음과 같다.

```text
URL
= 지금 어떤 상태인가

Event
= 어떤 행동으로 그 상태에 도달했는가
```

따라서 단순 복사 버튼 클릭을 다음처럼 만들지 않는다.

```text
?action=account.copy
```

복사 후에도 UI 상태가 바뀌지 않는다면 기존 Action은 유지하고 Analytics Event만 발생시킨다.

---

# 6. 민감정보 처리

## 6.1 기본 원칙

URL에 민감한 식별정보를 포함해야 하는 경우 payload를 암호화한다.

개념적인 처리 과정:

```text
Object
 ↓
JSON Serialize
 ↓
UTF-8
 ↓
Authenticated Encryption
 ↓
ciphertext + nonce + authentication data
 ↓
Base64URL
```

URL에 사용하는 문자열은 일반 Base64가 아니라 URL-safe encoding을 사용한다.

---

## 6.2 암호화 대상 예

상황에 따라 다음 값은 암호화된 payload에 포함할 수 있다.

```text
전화번호 관련 식별정보
내부 entity ID
계좌 조회 대상 식별정보
방명록 message ID
내부 resource ID
```

---

## 6.3 URL에 넣지 않는 Secret

암호화 여부와 관계없이 다음 값은 URL에 저장하지 않는다.

```text
사용자 비밀번호
Access Token
Refresh Token
API Credential
Private Key
세션 인증 Secret
```

암호화되어 있더라도 URL은 브라우저 History 및 각종 로그에 남을 수 있기 때문이다.

---

## 6.4 Payload Versioning

payload 포맷 앞에 버전을 명시할 수 있다.

```text
payload=v1.<encrypted-value>
```

예:

```ts
function decodePayload(value: string) {
  const [version, data] = splitPayload(value)

  switch (version) {
    case 'v1':
      return decodeV1(data)

    case 'v2':
      return decodeV2(data)

    default:
      throw new UnsupportedPayloadVersionError()
  }
}
```

향후 암호화 또는 직렬화 정책 변경에 대응할 수 있다.

---

# 7. Action Registry

기존의 Overlay Registry를 전체 UI Action Registry로 확장한다.

기존 설계에서는 route와 component, props, close policy, validation을 선언적으로 관리하는 `OverlayDefinition` 모델을 사용하고 있다.

본 설계에서는 이를 팝업에 한정하지 않는다.

```ts
interface ActionDefinition<TPayload = unknown> {
  action: string

  presentation:
    | 'page'
    | 'overlay'
    | 'sheet'
    | 'dialog'
    | 'inline'

  payload?: {
    schema: Schema<TPayload>
    encrypted?: boolean
  }

  navigation: NavigationPolicy

  validate?: (
    payload: TPayload
  ) => boolean
}
```

---

# 8. Presentation과 Navigation 분리

중요한 원칙이다.

```text
presentation
= 어떻게 화면에 표시되는가

navigation
= 브라우저 History에서 어떻게 동작하는가
```

따라서 다음이 가능하다.

```ts
{
  action: 'account.view',
  presentation: 'overlay',
  navigation: {
    enter: 'push',
    exit: 'back',
    lifetime: 'persistent'
  }
}
```

또 다른 Overlay는:

```ts
{
  action: 'photo.preview',
  presentation: 'overlay',
  navigation: {
    enter: 'push',
    exit: 'back',
    lifetime: 'ephemeral'
  }
}
```

두 UI 모두 Overlay지만 History 정책은 다르다.

---

# 9. Base Route + Query Overlay

본 설계는 `/before → /popup` 같은 Path 이동만 대상으로 하지 않는다.

주요 케이스는 기존 Route를 유지하면서 Query String으로 Overlay를 표현하는 방식이다.

예:

```text
/before
```

에서 계좌 팝업을 열면:

```text
/before?action=account.view&payload=...
```

가 된다.

브라우저 History 관점에서는:

```text
Entry A
/before

        ↓ push

Entry B
/before?action=account.view&payload=...
```

Path가 동일하더라도 별도의 History Entry다.

따라서 다음 두 전이는 같은 Navigation 시스템으로 관리한다.

```text
/before
→ /popup
```

```text
/before
→ /before?action=account.view...
```

Action Resolver는 Path가 아니라 Query의 `action`을 기준으로 어떤 추가 UI를 표시할지 판단한다.

---

# 10. Action Resolver

현재 Route를 UI 상태로 변환한다.

```text
route
 ↓
QueryStateParser
 ↓
payload decode/decrypt
 ↓
meta decode
 ↓
ActionRegistry lookup
 ↓
schema validate
 ↓
Navigation state check
 ↓
UI State Resolve
 ↓
Component Render
```

개념적 구현:

```ts
function resolveAction(route: RouteLocationNormalized) {
  const queryState = queryCodec.decode(route.query)

  if (!queryState.action) {
    return {
      presentation: 'base'
    }
  }

  const definition =
    actionRegistry[queryState.action]

  if (!definition) {
    return handleUnknownAction()
  }

  const payload =
    decodePayload(
      queryState.payload,
      definition.payload
    )

  validatePayload(
    definition,
    payload
  )

  return {
    definition,
    payload,
    meta: decodeMeta(queryState.meta)
  }
}
```

---

# 11. Action Dispatcher

컴포넌트는 Vue Router를 직접 조작하지 않는 것을 원칙으로 한다.

다음 코드는 지양한다.

```ts
router.push(...)
router.replace(...)
route.query...
```

대신:

```ts
const ui = useUiNavigation()

ui.dispatch('account.view', {
  target: 'groom'
})
```

또는:

```ts
ui.dispatch('guestbook.delete.auth', {
  messageId: id
})
```

형태를 사용한다.

Dispatcher 내부 흐름:

```text
Component
 ↓
ui.dispatch(action, payload)
 ↓
ActionRegistry lookup
 ↓
Payload validation
 ↓
Payload encode/encrypt
 ↓
Meta 생성
 ↓
NavigationPolicy 확인
 ↓
Router push/replace
 ↓
route.query 변경
 ↓
ActionResolver
 ↓
UI 변경
```

UI → URL과 URL → UI가 대칭 구조를 갖는다.

---

# 12. History Policy

## 12.1 기본 모델

```ts
interface NavigationPolicy {
  enter:
    | 'push'
    | 'replace'

  exit:
    | 'back'
    | 'restore'
    | 'replace'
    | 'custom'

  lifetime:
    | 'persistent'
    | 'ephemeral'

  directEntryExit?:
    | 'fallback'
    | 'replace'
    | 'custom'
}
```

---

# 13. Persistent Navigation

Persistent Action은 브라우저 History에 정상적인 탐색 상태로 남는다.

예:

```text
/before
 ↓

/before?action=account.view
```

History:

```text
[/before]
[/before?action=account.view]
                           ↑
```

닫을 때:

```text
router.back()
```

결과:

```text
[/before]
    ↑
[/before?action=account.view]
```

Forward하면 팝업이 다시 열린다.

즉:

```text
persistent
= Back으로 벗어난 상태가 Forward로 다시 복원될 수 있음
```

이다.

사용자가 브라우저 탐색 단계로 인식할 가치가 있는 UI에 사용한다.

---

# 14. Ephemeral Navigation

## 14.1 정의

Ephemeral은 다음 의미를 가진다.

```text
한 번 Back 또는 Close로 소비된 UI 상태는
같은 History Entry를 통해 다시 활성화하지 않는다.
```

단순히 `replace`만 사용하는 상태를 의미하지 않는다.

---

## 14.2 동작

초기 상태:

```text
/before
```

Ephemeral Overlay Open:

```text
/before
→ /before?action=photo.preview...
```

History:

```text
Entry A
/before

Entry B
/before?action=photo.preview...
                                ↑
```

사용자가 Back:

```text
Entry A
/before
   ↑

Entry B
/before?action=photo.preview...
```

브라우저 내부에는 Entry B가 Forward History로 남는다.

그러나 Navigation Layer에서 해당 Navigation Entry를 소비 상태로 만든다.

```text
navigationId = "nav-102"
consumed = true
```

사용자가 Forward를 누르면 브라우저 자체는 Entry B로 이동한다.

```text
/before?action=photo.preview...
```

하지만 Action Resolver는 해당 navigation이 이미 consumed 되었음을 확인한다.

```ts
if (
  navigation.lifetime === 'ephemeral' &&
  navigation.consumed
) {
  return restoreParentState()
}
```

따라서 Overlay를 다시 렌더링하지 않고 Parent 상태로 normalize한다.

사용자 관점에서는:

```text
Back으로 닫은 Ephemeral Overlay는
Forward로 다시 살아나지 않는다.
```

---

# 15. Ephemeral의 목적

브라우저 History API는 중간 History Entry를 직접 삭제할 수 없다.

따라서:

```text
/before → /popup
```

에서 Back한 후 `/popup`만 물리적으로 제거하는 방식에 의존하지 않는다.

대신:

```text
Physical Browser History
≠
Logical UI History
```

로 분리한다.

브라우저 History에는 Entry가 남아 있을 수 있지만 UI Navigation Layer에서는 해당 Entry를 이미 소비된 것으로 판단한다.

---

# 16. Navigation ID와 상태 추적

각 Navigation Entry는 내부적으로 고유 ID를 가질 수 있다.

예:

```ts
interface UiNavigationState {
  navigationId: string

  parentNavigationId?: string

  action?: string

  lifetime:
    | 'persistent'
    | 'ephemeral'

  consumed?: boolean

  openedFromApp?: boolean
}
```

예:

```text
nav-100
action = home

    ↓

nav-101
action = guestbook.view

    ↓

nav-102
action = photo.preview
lifetime = ephemeral
```

Back으로 `nav-101`로 이동하면:

```text
nav-102.consumed = true
```

로 취급한다.

---

# 17. history.state 활용

Query String에는 복원 가능한 UI 상태만 저장한다.

Navigation 내부 정보는 `history.state`를 활용할 수 있다.

예:

```ts
router.push({
  query: encodedQuery,

  state: {
    ui: {
      navigationId: 'nav-102',
      parentNavigationId: 'nav-101',
      entryType: 'overlay',
      lifetime: 'ephemeral',
      openedFromApp: true
    }
  }
})
```

구분은 다음과 같다.

```text
Query String
────────────────────
action
payload
meta

= 공유/복원 가능한 상태


history.state
────────────────────
navigationId
parentNavigationId
openedFromApp
lifetime

= 현재 브라우저 Session의 Navigation 정보
```

---

# 18. Direct Entry 처리

사용자가 앱 내부에서 다음 전이를 했다고 가정한다.

```text
/before
→ /before?action=account.view...
```

이 경우 Overlay Close는 `back()`으로 처리할 수 있다.

그러나 사용자가 외부 링크를 통해 바로:

```text
/before?action=account.view...
```

로 들어올 수도 있다.

이 경우 무조건 `router.back()`하면 외부 사이트로 빠질 수 있다.

따라서 다음 정책을 사용한다.

```text
앱 내부에서 Overlay Open
→ exit = back

URL 직접 진입
→ directEntryExit = fallback
```

예:

```ts
{
  action: 'account.view',

  navigation: {
    enter: 'push',
    exit: 'back',
    lifetime: 'persistent',
    directEntryExit: 'fallback'
  }
}
```

fallback 예:

```text
/
```

또는 Action별 Parent 상태.

---

# 19. Popup / Overlay Close 정책

Overlay 자체에서 Router를 직접 호출하지 않는다.

```ts
router.back()
router.replace('/')
```

를 기능 컴포넌트에 직접 작성하지 않는다.

대신:

```ts
ui.close()
```

만 호출한다.

Navigation Manager가 현재 ActionDefinition을 보고 종료 방식을 결정한다.

개념적 처리:

```ts
function closeCurrentAction() {
  const current = navigation.current
  const definition =
    actionRegistry[current.action]

  if (isDirectEntry(current)) {
    return handleDirectEntryExit(
      definition
    )
  }

  switch (definition.navigation.exit) {
    case 'back':
      return router.back()

    case 'restore':
      return restoreParent()

    case 'replace':
      return replaceWithParent()

    case 'custom':
      return definition.onExit?.()
  }
}
```

---

# 20. 브라우저 Back 감지

Router의 navigation hook 또는 필요한 범위에서 `popstate` 기반으로 Back/Forward 전이를 감지한다.

Back이 발생했을 때 현재 떠났던 Navigation Entry가 Ephemeral이라면 이를 consumed 처리한다.

개념:

```text
Current Navigation
nav-102
photo.preview
ephemeral

       ↓ browser back

Parent Navigation
nav-101

       ↓

NavigationManager

       ↓

nav-102 consumed 처리
```

이 처리는 Router Hook의 무한 재진입을 방지하도록 중앙 Navigation Manager에서만 수행한다.

---

# 21. Forward 처리

Forward로 들어온 URL이 이미 소비된 Ephemeral Navigation인지 검사한다.

```text
Browser Forward
 ↓
Route 변경
 ↓
ActionResolver
 ↓
Navigation ID 확인
 ↓
Consumed Ephemeral?
 ├─ No  → 정상 렌더
 └─ Yes → Parent State Restore
```

예:

```ts
if (
  navigationState.lifetime === 'ephemeral' &&
  navigationStore.isConsumed(
    navigationState.navigationId
  )
) {
  return normalizeToParent()
}
```

---

# 22. Action 상태 전이

방명록 생성 예:

```text
guestbook.view
      ↓
guestbook.create.message
      ↓
guestbook.create.name
      ↓
guestbook.create.password
      ↓
guestbook.view
```

각 단계가 사용자에게 독립적인 상태라면 각 Action을 History에 `push`할 수 있다.

그러면 브라우저 Back으로:

```text
password
→ name
→ message
→ guestbook.view
```

와 같은 전이가 가능하다.

반대로 중간 단계가 Browser History에 남을 필요가 없다면 `replace` 정책을 사용할 수 있다.

---

# 23. 팝업 안의 중첩 다이얼로그

예:

```text
guestbook.create.message
```

작성 도중 닫기:

```text
guestbook.create.discard-confirm
```

이 Dialog는 다음과 같이 정의할 수 있다.

```ts
{
  action:
    'guestbook.create.discard-confirm',

  presentation: 'dialog',

  navigation: {
    enter: 'push',
    exit: 'back',
    lifetime: 'ephemeral'
  }
}
```

사용자가 취소하거나 Back으로 닫은 Confirm Dialog가 Forward로 다시 나타나지 않게 만들 수 있다.

---

# 24. OverlayHost

기존 설계의 `OverlayHost` 구조는 그대로 유지할 수 있다.

기존 구조는 OverlayHost가 어떤 팝업을 렌더할지 결정하고 `ModalShell`이 공통 UI 제어를 담당한다.

본 설계에서는 OverlayHost가 `route.name`이나 개별 Query Key를 직접 판단하는 대신 ActionResolver의 결과를 사용한다.

```text
App.vue
├─ Base Content
├─ Global Actions
└─ OverlayHost
     └─ ModalShell
          └─ Dynamic Action Component
```

흐름:

```text
route.query
 ↓
ActionResolver
 ↓
ActionDefinition
 ↓
presentation 확인
 ↓
overlay / dialog / sheet?
 ↓
OverlayHost
 ↓
ModalShell
 ↓
Dynamic Component
```

---

# 25. ModalShell의 책임

ModalShell은 업무 로직을 갖지 않는다.

담당 범위:

```text
backdrop
ESC
scroll lock
focus trap
focus restore
transition
z-index
close request
```

팝업 내부 컴포넌트는 다음에만 집중한다.

```text
표시 데이터
입력 데이터
API 요청
복사/전화/문자 등 도메인 기능
닫기 가능 여부
미저장 상태 여부
```

기존 설계의 ModalShell 책임과 동일한 방향이다.

---

# 26. Section + Teleport

기능 상태를 특정 Section이 소유해야 하는 경우 해당 로직은 Section에 유지한다.

예:

```text
ContactSection
├─ 사람 목록
├─ 연락처 데이터
├─ 기능 로직
└─ Teleport
     └─ Overlay DOM
```

실제 DOM은 다음 위치에 둘 수 있다.

```text
body
└─ #overlay-root
```

기존 설계에서도 Section의 응집도를 유지하면서 Overlay DOM만 전역 레이어로 이동시키는 방식이 제안되어 있다.

다만 Action의 상태 기준은 Section 내부 ref가 아니라 Query Action이다.

---

# 27. UI Action Naming Convention

Action Name은 다음 형식을 권장한다.

```text
<domain>.<operation>[.<state>]
```

예:

```text
account.view

contact.view

chat.open

guestbook.view

guestbook.create.message
guestbook.create.name
guestbook.create.password

guestbook.modify.auth
guestbook.modify.edit

guestbook.delete.auth
guestbook.delete.confirm

directions.view
directions.map.view

photo.preview
```

버튼명, 함수명, Vue 컴포넌트명은 Action 이름에 사용하지 않는다.

---

# 28. 예시 Action Registry

```ts
const actionRegistry = {

  'account.view': {
    presentation: 'overlay',

    payload: {
      schema: AccountPayloadSchema,
      encrypted: true
    },

    navigation: {
      enter: 'push',
      exit: 'back',
      lifetime: 'persistent',
      directEntryExit: 'fallback'
    }
  },

  'contact.view': {
    presentation: 'overlay',

    payload: {
      schema: ContactPayloadSchema,
      encrypted: true
    },

    navigation: {
      enter: 'push',
      exit: 'back',
      lifetime: 'persistent',
      directEntryExit: 'fallback'
    }
  },

  'photo.preview': {
    presentation: 'overlay',

    payload: {
      schema: PhotoPayloadSchema
    },

    navigation: {
      enter: 'push',
      exit: 'back',
      lifetime: 'ephemeral',
      directEntryExit: 'fallback'
    }
  },

  'guestbook.create.message': {
    presentation: 'overlay',

    navigation: {
      enter: 'push',
      exit: 'back',
      lifetime: 'persistent',
      directEntryExit: 'fallback'
    }
  },

  'guestbook.delete.confirm': {
    presentation: 'dialog',

    payload: {
      schema: GuestbookTargetSchema,
      encrypted: true
    },

    navigation: {
      enter: 'push',
      exit: 'back',
      lifetime: 'ephemeral',
      directEntryExit: 'fallback'
    }
  }
}
```

---

# 29. 전체 실행 흐름

## 29.1 UI Action 발생

```text
사용자
 ↓
버튼 클릭
 ↓
Component
 ↓
ui.dispatch(
  "account.view",
  payload
)
```

---

## 29.2 Navigation 생성

```text
ActionRegistry 조회
 ↓
Payload Schema Validation
 ↓
Payload 암호화/직렬화
 ↓
Meta 구성
 ↓
Navigation ID 생성
 ↓
Navigation Policy 확인
 ↓
router.push / router.replace
```

---

## 29.3 Route 변경

```text
route.query 변경
 ↓
QueryStateParser
 ↓
ActionResolver
 ↓
Payload Decode
 ↓
Validation
 ↓
Navigation Lifetime 검사
 ↓
ActionDefinition 반환
```

---

## 29.4 렌더링

```text
presentation = page
→ Router View

presentation = overlay
→ OverlayHost

presentation = dialog
→ Dialog Host

presentation = sheet
→ OverlayHost / SheetShell

presentation = inline
→ 해당 Section
```

---

# 30. 닫기 흐름

```text
사용자
 ↓
Close Button / ESC / Backdrop
 ↓
ModalShell
 ↓
requestClose(reason)
 ↓
UiNavigationManager
 ↓
미저장 상태 확인
 ↓
ActionDefinition.navigation.exit 확인
 ↓
back / restore / replace / custom
 ↓
route 변경
 ↓
Overlay leave transition
 ↓
scroll unlock
 ↓
focus restore
```

기존 설계에서도 닫기 사유와 close policy를 중앙에서 관리하도록 되어 있다.

---

# 31. 잘못된 URL 처리

다음과 같은 URL이 들어올 수 있다.

```text
?action=unknown.action
```

또는:

```text
?action=account.view
&payload=<invalid-payload>
```

처리 흐름:

```text
Query Parse
 ↓
Action Exists?
 ├─ No
 │   ↓
 │ fallback
 │
 └─ Yes
     ↓
 Payload Decode
     ↓
 Schema Valid?
 ├─ No
 │   ↓
 │ fallback / error state
 │
 └─ Yes
     ↓
 정상 렌더
```

기존 설계에서도 route query를 중앙에서 검증하고 유효하지 않은 URL을 home으로 replace하거나 오류 UI로 처리하도록 권장하고 있다.

---

# 32. QA / E2E 테스트

본 구조에서는 특정 UI 상태를 URL로 직접 재현할 수 있다.

예:

```ts
await page.goto(
  '/?action=guestbook.create.name&payload=...'
)
```

또는:

```ts
await page.goto(
  '/?action=guestbook.delete.auth&payload=...'
)
```

테스트 시 초기 단계부터 UI를 반복 실행할 필요 없이 특정 중간 상태로 직접 진입할 수 있다.

QA에서도:

```text
/action URL 전달
```

만으로 동일한 상태를 재현할 수 있다.

---

# 33. 컴포넌트 구현 규칙

## 금지

일반 Feature Component 내부에서 다음 패턴을 남발하지 않는다.

```ts
router.push(...)
router.replace(...)
route.query.action
```

또한:

```ts
isPopupOpen.value = true
selectedStep.value = 'password'
```

등을 Navigable UI State의 Source of Truth로 사용하지 않는다.

---

## 권장

```ts
const ui = useUiNavigation()

ui.dispatch(
  'guestbook.delete.auth',
  {
    messageId
  }
)
```

현재 상태 조회 역시:

```ts
const state =
  useResolvedUiState()
```

를 통해 접근한다.

---

# 34. 권장 모듈 구조

```text
src/
├─ navigation/
│  ├─ action-registry.ts
│  ├─ action-types.ts
│  ├─ query-codec.ts
│  ├─ payload-codec.ts
│  ├─ meta-codec.ts
│  ├─ action-resolver.ts
│  ├─ navigation-manager.ts
│  ├─ navigation-store.ts
│  └─ use-ui-navigation.ts
│
├─ overlay/
│  ├─ OverlayHost.vue
│  ├─ ModalShell.vue
│  ├─ DialogHost.vue
│  └─ overlay-root.ts
│
├─ features/
│  ├─ account/
│  ├─ contact/
│  ├─ chat/
│  ├─ directions/
│  └─ guestbook/
│
└─ router/
   └─ index.ts
```

---

# 35. 주요 인터페이스 예시

```ts
interface UiActionPayload {
  [key: string]: unknown
}

interface UiActionMeta {
  v?: number
  from?: string
  presentation?: string
}

interface QueryState {
  action?: string
  payload?: string
  meta?: string
}

interface NavigationPolicy {
  enter:
    | 'push'
    | 'replace'

  exit:
    | 'back'
    | 'restore'
    | 'replace'
    | 'custom'

  lifetime:
    | 'persistent'
    | 'ephemeral'

  directEntryExit?:
    | 'fallback'
    | 'replace'
    | 'custom'
}

interface UiNavigationState {
  navigationId: string
  parentNavigationId?: string

  action?: string

  lifetime:
    | 'persistent'
    | 'ephemeral'

  consumed?: boolean
  openedFromApp?: boolean
}

interface ActionDefinition<TPayload = unknown> {
  action: string

  presentation:
    | 'page'
    | 'overlay'
    | 'sheet'
    | 'dialog'
    | 'inline'

  payload?: {
    schema: Schema<TPayload>
    encrypted?: boolean
  }

  navigation: NavigationPolicy

  validate?: (
    payload: TPayload
  ) => boolean
}
```

---

# 36. 전체 아키텍처

```text
                         URL
                          │
            ┌─────────────┴─────────────┐
            │                           │
          path                       query
                                   action
                                   payload
                                   meta
            │                           │
            └─────────────┬─────────────┘
                          ▼
                  QueryStateParser
                          │
                          ▼
                    PayloadCodec
                  decode / decrypt
                          │
                          ▼
                   ActionResolver
                          │
                ActionRegistry Lookup
                          │
                  Schema Validation
                          │
                Navigation Validation
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
        Page           Overlay           Dialog
                          │
                          ▼
                    OverlayHost
                          │
                          ▼
                     ModalShell
                          │
                          ▼
                   Feature Component


사용자 Action
      │
      ▼
ui.dispatch(action, payload)
      │
      ▼
ActionRegistry
      │
      ▼
NavigationManager
      │
      ├─ Payload encode/encrypt
      ├─ Meta encode
      ├─ NavigationId
      ├─ History Policy
      └─ Analytics
      │
      ▼
Vue Router
      │
      └────────────────────────────→ URL
```

---

# 37. 최종 설계 원칙 요약

```text
1.
URL에는 현재 사용자가 보고 있는
의미 있는 UI 상태를 저장한다.

2.
Query Key는
action / payload / meta
세 개로 일원화한다.

3.
action은 UI 상태의 Identity다.

4.
payload는 해당 상태가 필요로 하는 데이터다.

5.
meta는 상태 자체가 아닌 부가 문맥이다.

6.
민감 데이터 payload는 암호화한다.

7.
Secret은 암호화 여부와 관계없이
URL에 넣지 않는다.

8.
Component는 Router를 직접 조작하지 않고
ui.dispatch()를 통해 상태를 전이한다.

9.
ActionRegistry가
UI 상태와 Component,
Presentation,
Navigation Policy를 연결한다.

10.
presentation과 history 정책은 분리한다.

11.
Overlay는 별도의 Route Path가 아니어도 된다.

/before
→ /before?action=...

형태를 1급 Navigation으로 취급한다.

12.
Persistent Navigation은
Back 후 Forward로 복원될 수 있다.

13.
Ephemeral Navigation은
Back 또는 Close로 한 번 소비된 이후
같은 Navigation Entry를 Forward로 다시
활성화하지 않는다.

14.
브라우저 History Entry 자체를 억지로 삭제하지 않는다.

Physical Browser History와
Logical UI History를 분리한다.

15.
팝업 Close는 Component가 직접
router.back() 또는 router.replace()를
선택하지 않는다.

UiNavigationManager가 Action별
Navigation Policy에 따라 결정한다.

16.
직접 URL 접근과
앱 내부 Navigation을 구분한다.

17.
Navigable UI State,
Domain Session State,
UI Infrastructure State,
Analytics Event를 서로 분리한다.
```

본 구조의 핵심은 기존 설계에서 정의한 다음 원칙을 애플리케이션 전체로 확장하는 것이다.

```text
라우트는 “무엇을 열었는가”를 담당하고,
기능 컴포넌트는 “그 안에서 무엇을 하는가”를 담당하며,
공통 오버레이 계층은
“어떻게 안전하고 일관되게 화면을 덮는가”를 담당한다.
```

기존 문서에서도 동일한 역할 분리가 핵심 원칙으로 정리되어 있다.

본 설계에서는 여기에 `ActionRegistry`, `NavigationPolicy`, `Persistent/Ephemeral Lifetime`, `NavigationId`를 추가함으로써 팝업뿐 아니라 페이지, 다이얼로그, 단계형 UI, Query 기반 Overlay까지 하나의 URL-driven UI State Machine으로 관리한다.
