# 🔄 Gem Lifecycle

```text
Mine → Refine → Stash → Socket → Craft → Validate → Upgrade
 ↑                                                        │
 └────────────────────────────────────────────────────────┘
```

## Mine

실제로 동작한 시스템에서 재사용 가치가 있는 책임·계약·실패 정책·보안 판단을 찾습니다.

## Refine

프로젝트명, 내부 경로, 업무 고유 Naming, 우연한 구현 Detail을 제거합니다. 대신 구현에 필요한 Engineering Intent와 Constraint를 남깁니다.

## Stash

하나의 Gem을 하나의 디렉터리로 저장하고 metadata로 검색 가능한 정보를 부여합니다.

## Socket

새 프로젝트의 책임 영역과 Gem의 책임이 맞는지 확인합니다. Framework나 언어는 달라도 괜찮습니다.

## Craft

여러 Gem 사이의 Contract가 맞지 않으면 기존 Gem을 억지로 합치기보다 Adapter와 Glue 설계를 별도로 만듭니다.

## Validate

Schema, Contract, Security Rule, Automated Test 등으로 생성된 구현의 허용 경계를 검사합니다.

## Upgrade

실제 사용에서 발견한 새로운 판단을 기존 Gem에 반영하거나 새로운 Mixed Gem으로 분리합니다.
