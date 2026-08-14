# Structured UI Contract — Backend

## Intent

LLM은 HTML, Script, Component Import Path, 함수 또는 임의 API Endpoint를 만들지 않습니다. 사전에 정의된 Component, Props, Action Contract 안에서만 응답합니다.

## Response Contract

```text
version
component
props
actions
metadata
```

`metadata`의 requestId, model/provider, contractVersion 등은 LLM이 아니라 Backend가 생성합니다.

## Allowlist

Backend는 다음을 Allowlist로 관리합니다.

- Component
- Component별 Props Schema
- Action
- Action별 Payload Schema
- Component ↔ Action 허용 관계

Schema상 유효하더라도 해당 Component에서 허용하지 않은 Action은 거부합니다.

## Workflow Boundary

LLM은 외부 작업을 **제안**할 수 있지만 실행하지 않습니다.

```text
LLM Response
  ↓ proposes action
Frontend explicit user action
  ↓
Execution API
  ↓ validate again
External side effect
```

성공 UI는 실제 Side Effect 성공 결과를 기반으로 Backend/Frontend가 생성하며, LLM이 “성공했다”고 말한 것만으로 표시하지 않습니다.

## Runtime Validation

LLM Output 직후 다음을 검증합니다.

- JSON parse
- contract version
- component enum
- props schema
- action enum / payload
- component-action policy
- response size / string length
- state/workflow policy where applicable

실패한 원본 응답을 Client로 그대로 전달하지 않고 안전한 Fallback Contract로 변환합니다.

## Prompt Boundary

System Rules, UI Contract, Component/Action Definition, Runtime State, User Conversation, Output Schema를 명확히 분리합니다. 사용자 입력이 Schema 무시, 미등록 Component/Action, System Prompt 공개를 요구해도 계약을 바꾸지 않습니다.

## Security

- `eval`, Function Constructor, dynamic import from model output 금지
- arbitrary endpoint / script / HTML 금지
- external link는 등록 ID + allowlist 사용
- private input/prompt/payload를 기본 로그에서 제외
