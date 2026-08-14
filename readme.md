```text
 _   _  ___  _   _  ____    _  _____
| \ | |/ _ \| | | |/ ___|  / \|_   _|
|  \| | | | | | | | |  _  / _ \ | |
| |\  | |_| | |_| | |_| |/ ___ \| |
|_| \_|\___/ \___/ \____/_/   \_\_|

 ____  _   _ ___ ____  ____  _____ _____
/ ___|| \ | |_ _|  _ \|  _ \| ____|_   _|
\___ \|  \| || || |_) | |_) |  _|   | |
 ___) | |\  || ||  __/|  __/| |___  | |
|____/|_| \_|___|_|   |_|   |_____| |_|
```

# 💎 nougat-snippet

> **Mine proven software. Refine the decisions. Socket the Gems. Regenerate the code.**

`nougat-snippet`은 단순한 코드 스니펫 저장소를 넘어, 실제 프로젝트에서 얻은 **재사용 가능한 Engineering Decision**을 모으는 Gem Stash를 지향합니다.

AI가 코드를 다시 만들 수 있는 시대라면, 예전 프로젝트의 코드를 그대로 복사하는 것보다 그 코드를 좋게 만든 **설계·책임·계약·실패 정책·보안 판단**을 남기는 편이 더 오래 재사용할 수 있다고 생각합니다.

이 방식을 이 저장소에서는 **Gem Programming**이라고 부릅니다.

## ⛏️ Gem Programming

게임에서 좋은 장비를 통째로 복사하기보다 장비 안의 보석을 꺼내 새 장비에 다시 꽂듯이, 기존 프로젝트에서 재사용할 가치가 있는 설계를 추출하고 새로운 프로젝트의 Context에 맞게 다시 구현합니다.

```text
Existing Project
      ↓
     Mine
      ↓
   Raw Gem
      ↓
    Refine
      ↓
     Gem
      ↓
    Stash
      ↓
 New Project
      ↓
   Socket
      ↓
   Craft
      ↓
  Generate
      ↓
  Validate
      ↓
 Mine Again
```

핵심은 **같은 코드를 재사용하는 것보다 같은 Engineering Intent와 Constraint를 재사용하는 것**입니다.

```text
Code reuse moves implementations.
Gem Programming moves engineering decisions.
```

## 💎 Gem이란?

Gem은 실제 구현 경험에서 추출·정제한, 다른 프로젝트에서도 LLM이나 AI Agent가 다시 구현할 수 있을 만큼 구체적인 설계 단위입니다.

일반적인 Design Pattern보다는 구체적이지만 특정 언어·프레임워크의 Code에는 묶이지 않는 수준을 목표로 합니다.

Gem에는 상황에 따라 다음과 같은 내용이 포함될 수 있습니다.

- 해결하려는 문제와 Context
- 책임과 시스템 경계
- 입력·출력 Contract
- Data Flow와 상태 전이
- Failure / Recovery Policy
- Security Constraint
- Validation Rule
- 주요 Trade-off와 선택 이유

즉, 코드를 요약하는 것이 아니라 **그 코드를 좋게 만든 Engineering Decision을 증류해 남기는 것**이 목적입니다.

## 🗃️ 이 저장소는 Gem Stash입니다

이 저장소에는 크게 두 종류의 Gem이 쌓입니다.

### Origin Gem

실제로 동작한 프로젝트나 구현에서 독립적으로 추출한 설계입니다.

```text
Working Project
      ↓ Mine
   Origin Gem
```

예를 들어 Streaming Proxy, LLM UI, 암복호화처럼 하나의 문제 영역에서 검증된 설계를 추출합니다.

### Mixed / Integration Gem

서로 다른 Origin Gem을 함께 사용하면서 새롭게 발견한 Adapter, Contract, Validation, Recovery 정책 등을 정리합니다.

```text
Origin Gem A
      +
Origin Gem B
      ↓
 Contract Gap
      ↓
   Crafting
      ↓
Integration Gem C
```

좋은 Gem 두 개가 있다고 해서 항상 바로 연결되는 것은 아닙니다. 서로 다른 설계를 조합하는 과정에서 새로운 Engineering Knowledge가 생기면, 그 지식 역시 다시 하나의 Gem으로 남깁니다.

## 🔌 사용하는 방법

새 프로젝트를 시작할 때 이 저장소의 코드를 그대로 복사하는 것을 기본 사용법으로 생각하지 않습니다.

먼저 해결하려는 영역에 맞는 Gem을 찾고, 현재 프로젝트의 기술 스택과 제약조건을 함께 제공한 뒤 구현을 다시 생성하는 방식을 권장합니다.

```text
Gem Stash
    ↓
Gem + Current Context
    ↓
LLM / AI Agent
    ↓
Context-specific Implementation
    ↓
Validation
```

같은 Gem을 NestJS, Go, Rust 등 서로 다른 환경에 적용하면 결과 코드는 달라질 수 있습니다.

중요한 것은 코드의 모양이 아니라 **Gem에 정의된 Contract와 Constraint가 유지되는가**입니다.

## 🛡️ LLM을 그대로 믿지는 않습니다

LLM은 비결정적이기 때문에 같은 Gem에서도 구현 결과가 달라질 수 있습니다.

Gem Programming에서는 항상 같은 코드를 만들게 하기보다, 결과가 반드시 지켜야 할 경계를 명확히 만드는 방향을 지향합니다.

```text
              LLM
               ↓
      Different Implementations
               ↓
┌─────────────────────────┐
│ Gem Constraints         │
│ Socket Contract         │
│ Schema                  │
│ Security Rules          │
│ Validation              │
│ Automated Tests         │
└────────────┬────────────┘
             ↓
       Accepted Result
```

구현에는 자유를 주되, 시스템의 규칙을 깨는 결과는 통과시키지 않는 것이 목표입니다.

## 🚧 아직 초안입니다

Gem Programming은 완성된 방법론이 아닙니다.

현재 실제 프로젝트에서 사용하면서 Gem의 적절한 크기, 문서 구조, Origin과 Mixed Gem의 관계, LLM이 안정적으로 구현하기 위해 필요한 정보의 수준 등을 계속 정리하고 있습니다.

따라서 이 저장소의 구조와 규칙도 앞으로 바뀔 수 있습니다.

더 나은 추상화 방식이나 문서 구조가 있다면 Issue로 의견을 남겨주세요. 새로운 Gem, 기존 Gem의 개선, 구조 변경 제안 등 **PR도 환영합니다.**

이 저장소 자체도 아래 Loop를 따라 계속 발전시키려고 합니다.

```text
Mine → Refine → Stash → Socket → Craft → Validate → Upgrade
  ↑                                                        │
  └────────────────────────────────────────────────────────┘
```

## ✨ 원칙

좋은 프로젝트는 하나의 완성품이면서 동시에 다음 프로젝트를 위한 광산이라고 생각합니다.

프로젝트는 끝나고 Framework는 바뀌지만, 그 안에서 어렵게 얻은 좋은 판단까지 사라질 필요는 없습니다.

**검증된 프로젝트에서 보석을 캐고, 좋은 판단만 다듬어 남기고, 새로운 프로젝트에 꽂고 조합해 코드는 다시 만듭니다.**
