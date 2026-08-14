# Structured UI Contract — Frontend

## Architecture

```text
API Response
  ↓ decrypt if applicable
Runtime Contract Validation
  ↓
Normalize
  ↓
Message / View State
  ↓
Component Registry
  ↓
Renderer

User Event
  ↓
Action Dispatcher
  ↓
Action Registry
  ↓
Payload Validation
  ↓
State Guard
  ↓
Handler
```

## Component Registry

Backend의 문자열을 Component 이름이나 Import Path로 직접 실행하지 않습니다. 명시적으로 등록된 Component만 선택합니다.

Registry는 Component 선택만 담당하고 API 호출, State Mutation, Action 실행을 직접 하지 않습니다.

## Action Dispatcher

UI Component는 Event만 Emit합니다. Dispatcher가 Action 존재 여부, Payload Schema, 현재 State와 Transition Guard를 확인한 뒤 Handler를 실행합니다.

## State Machine

외부 Side Effect가 있는 Workflow는 명시적인 상태 전이를 두는 것이 좋습니다.

```text
idle → editing → ready → executing → completed
                          ↘ error
```

중복 실행, 필수값 누락, 성공 전 완료 상태 진입 같은 전이를 Guard에서 차단합니다.

## Fallback

다음 오류는 해당 UI 범위에서 안전한 Fallback으로 격리합니다.

- contract version mismatch
- unknown component/action
- props/payload validation failure
- renderer error
- invalid state transition

Fallback에는 Stack Trace, 내부 Endpoint, Secret, Prompt 원문을 노출하지 않습니다.

## Security

- Raw HTML 기본 비활성화 또는 sanitization
- 위험 URL scheme 차단
- arbitrary iframe/script 금지
- external link allowlist
- 민감 Draft를 영구 Browser Storage에 자동 저장하지 않음
