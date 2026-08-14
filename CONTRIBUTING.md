# 🤝 Contributing

이 저장소는 아직 구조를 잡아가는 단계입니다. 작은 문서 개선부터 새로운 Gem 제안까지 PR을 환영합니다.

## Gem을 추가할 때

1. 실제로 동작했거나 충분히 검증한 구현에서 Engineering Decision을 추출합니다.
2. 프로젝트 고유 정보를 제거하고 재사용 가능한 수준으로 Refine합니다.
3. `templates/`의 형식을 참고해 `gems/origin` 또는 `gems/mixed` 아래에 추가합니다.
4. `metadata.yaml`에 `status`, `domains`, `targets`, `parents`를 명시합니다.
5. 구현 방법보다 Contract, Constraint, Failure Policy, Validation을 우선해 기록합니다.

## 🔒 Redaction Rules

PR 전에 다음 정보가 남아 있지 않은지 확인해 주세요.

- 비공개 프로젝트·고객·회사·조직 이름
- 개인 이름, 이메일, 전화번호, 계좌 등 식별정보
- 사설/개인 도메인과 실제 운영 API URL
- 로컬 절대경로 (`C:\\Users\\...`, `/home/...`, `/srv/...` 등)
- 실제 Repository 이름이나 내부 Workspace 경로
- Secret, API Key, Token, Private Key, Cookie의 실제 이름/값
- 내부 DB Table, Queue, Bucket, Host, Resource ID
- 프로젝트 구조를 그대로 복원할 수 있는 상세 파일 트리
- 운영 로그·Prompt·Payload의 실제 사용자 데이터

필요한 예시는 `example.com`, `/api/resource`, `example_session`, `resourceId`처럼 일반화된 이름을 사용합니다.

공개 라이브러리·표준·브라우저·프레임워크 이름은 기술적 의미가 있을 때 그대로 사용할 수 있습니다.

## Status

- `draft`: 구조를 잡는 단계
- `experimental`: 실제 적용 중이며 변경 가능성이 큼
- `validated`: 한 개 이상의 실제 구현에서 검증
- `stable`: 반복 적용하며 계약이 비교적 안정적
- `deprecated`: 더 이상 권장하지 않음

## PR Checklist

- [ ] 프로젝트 고유 정보와 개인정보를 제거했어요.
- [ ] 문제와 Engineering Intent가 명확해요.
- [ ] Contract와 Constraint를 구현 Detail보다 먼저 설명해요.
- [ ] Failure / Recovery Policy가 필요한 경우 포함했어요.
- [ ] LLM이 구현할 때 지켜야 할 Validation Rule이 있어요.
- [ ] 특정 Framework에만 필요한 Detail은 예시로 분리했어요.
- [ ] Mixed Gem이라면 Parent Gem을 명시했어요.
