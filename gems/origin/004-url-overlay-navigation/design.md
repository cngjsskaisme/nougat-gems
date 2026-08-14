# URL-based UI Action / Overlay Navigation

## 1. Intent

사용자가 인지하고 새로고침·딥링크·브라우저 Back/Forward로 복원할 가치가 있는 UI 상태를 URL로 표현합니다. 입력 Draft, Loading, Focus, Animation 같은 일시 상태는 URL에 넣지 않습니다.

## 2. Query Contract

```text
?action=<ACTION>&payload=<PAYLOAD>&meta=<META>
```

- `action`: 현재 UI 상태의 Identity
- `payload`: 해당 상태가 필요로 하는 대상 데이터
- `meta`: 버전, 유입 경로, presentation 같은 부가 문맥

예시는 업무 고유 이름 대신 일반화된 이름을 사용합니다.

```text
?action=profile.view
?action=item.preview&payload=...
?action=form.edit&payload=...
?action=confirm.discard
```

Component 이름이나 함수 이름을 Action ID로 사용하지 않습니다.

## 3. State Layers

```text
Navigable UI State  → URL
Domain Session State → feature-local state
UI Infrastructure   → overlay infrastructure
Analytics Event      → event pipeline
```

URL은 “지금 어디에 있는가”를 나타내고 Event는 “어떤 행동으로 도달했는가”를 나타냅니다.

## 4. Action Registry

```ts
interface ActionDefinition<TPayload = unknown> {
  action: string
  presentation: 'page' | 'overlay' | 'sheet' | 'dialog' | 'inline'
  payload?: {
    schema: Schema<TPayload>
    encrypted?: boolean
  }
  navigation: NavigationPolicy
}
```

Presentation은 어떻게 보이는지, Navigation은 History에서 어떻게 동작하는지를 결정합니다.

## 5. Navigation Policy

```ts
interface NavigationPolicy {
  enter: 'push' | 'replace'
  exit: 'back' | 'restore' | 'replace' | 'custom'
  lifetime: 'persistent' | 'ephemeral'
  directEntryExit?: 'fallback' | 'replace' | 'custom'
}
```

### Persistent

Back으로 벗어난 뒤 Forward하면 같은 UI 상태가 다시 복원될 수 있습니다.

### Ephemeral

Browser History Entry를 물리적으로 삭제하려 하지 않습니다. 대신 Logical UI History에서 “이미 소비된 상태”를 추적하고, Forward로 다시 들어왔을 때 Parent 상태로 Normalize합니다.

```text
Physical Browser History ≠ Logical UI History
```

## 6. Dispatcher / Resolver Symmetry

```text
Component
  ↓ ui.dispatch(action, payload)
Registry + Validation
  ↓ encode
Router
  ↓ URL
Resolver
  ↓ decode + validate
Resolved UI State
```

Feature Component는 `router.push()` / `router.back()` 정책을 직접 결정하지 않고 중앙 Navigation Manager에 의도를 전달합니다.

## 7. Direct Entry

앱 내부에서 Overlay를 열었다면 Close가 `back`일 수 있지만, 사용자가 Overlay URL로 직접 진입한 경우 `back`은 외부 사이트로 이동할 수 있습니다. 따라서 `directEntryExit` 정책을 별도로 둡니다.

## 8. Sensitive Payload

URL은 History, Log, Referrer 등에 남을 수 있습니다. Secret, Token, Password, Credential은 암호화 여부와 관계없이 URL에 넣지 않습니다.

민감한 식별자를 꼭 URL에 표현해야 한다면 최소화하고, Authenticated Encryption과 URL-safe encoding을 사용할 수 있습니다. 그래도 서버 측 권한 검증은 별도로 필요합니다.

## 9. Overlay Infrastructure

공통 Overlay Shell은 backdrop, ESC, scroll lock, focus trap/restore, transition, z-index와 close request를 담당합니다. Feature는 표시 데이터와 업무 상태에 집중합니다.

## 10. Validation

- unknown action → fallback
- payload decode/schema failure → fallback or safe error state
- direct entry와 in-app navigation 구분
- consumed ephemeral state의 Forward 복원 차단
- focus/scroll cleanup 중복 실행 방지
- URL에 Secret이 포함되지 않는지 검사
