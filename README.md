# 💎 nougat-snippet

> **Mine proven software. Refine the decisions. Socket the Gems. Regenerate the code.**

`nougat-snippet`은 실제 프로젝트에서 검증된 **Engineering Decision**을 재사용 가능한 설계 단위인 **Gem**으로 추출하고 보관하는 실험적 Gem Stash입니다.

코드를 그대로 복사하는 대신, 그 코드를 좋게 만든 책임·계약·실패 정책·보안 판단·상태 전이·Trade-off를 남기고 LLM/AI Agent가 현재 프로젝트의 Context에 맞게 다시 구현하도록 하는 방식을 **Gem Programming**이라고 부릅니다.

> 🚧 **Draft** — Gem Programming의 문서 형식과 Repository 구조는 실제 사용을 통해 계속 다듬는 중입니다. 더 나은 구조나 Gem이 있다면 Issue와 PR을 환영합니다.

## ⛏️ Game Loop

```text
Existing Project
      ↓ Mine
   Raw Gem
      ↓ Refine
     Gem
      ↓ Stash
 New Project
      ↓ Socket
    Craft
      ↓
Context-specific Code
      ↓ Validate
 Better Knowledge
      └────────→ Mine Again
```

**Code reuse moves implementations. Gem Programming moves engineering decisions.**

## 🗃️ Repository Structure

```text
.
├─ README.md
├─ CONTRIBUTING.md
├─ gems/
│  ├─ origin/
│  │  ├─ 001-streaming-proxy/
│  │  ├─ 002-llm-ui/
│  │  ├─ 003-hybrid-encryption/
│  │  ├─ 004-url-overlay-navigation/
│  │  ├─ 005-mobile-native-fab-avoidance/
│  │  ├─ 006-i18n-architecture/
│  │  ├─ 007-guest-hmac-auth/
│  │  └─ 008-structured-ui-contract/
│  └─ mixed/
│     └─ 001-002-ndjson-json-render/
├─ templates/
├─ docs/
└─ examples/
```

하나의 Gem을 하나의 디렉터리로 관리합니다. FE/BE처럼 서로 다른 실행 영역이 있더라도 동일한 Engineering Intent를 공유하면 같은 Gem 아래에 둡니다.

## 💎 Current Gems

| ID | Gem | Type | Status |
|---|---|---|---|
| 001 | [Streaming Proxy](./gems/origin/001-streaming-proxy/) | Origin | Draft |
| 002 | [LLM UI / json-render](./gems/origin/002-llm-ui/) | Origin | Draft |
| 003 | [Hybrid Encryption](./gems/origin/003-hybrid-encryption/) | Origin | Draft |
| 004 | [URL Overlay Navigation](./gems/origin/004-url-overlay-navigation/) | Origin | Draft |
| 005 | [Mobile Native FAB Avoidance](./gems/origin/005-mobile-native-fab-avoidance/) | Origin | Draft |
| 006 | [i18n Architecture](./gems/origin/006-i18n-architecture/) | Origin | Draft |
| 007 | [Guest HMAC Authentication](./gems/origin/007-guest-hmac-auth/) | Origin | Draft |
| 008 | [Structured UI Contract](./gems/origin/008-structured-ui-contract/) | Origin | Draft |
| 001+002 | [NDJSON × json-render Adapter](./gems/mixed/001-002-ndjson-json-render/) | Mixed | Draft |

## 🔌 How to Use a Gem

Gem은 복사할 코드가 아니라 **다시 구현할 설계**입니다.

```text
Gem
 + Current Stack
 + Project Constraints
 + Validation Rules
        ↓
    LLM / Agent
        ↓
Implementation
        ↓
Tests / Contract / Security Validation
```

같은 Gem을 NestJS, Go, Rust 또는 다른 환경에 적용하면 코드 모양은 달라질 수 있습니다. 중요한 것은 Gem에 정의한 Engineering Intent와 Constraint가 유지되는지입니다.

## 🧬 Origin과 Mixed

**Origin Gem**은 하나의 검증된 구현에서 독립적으로 추출한 설계입니다.

**Mixed Gem**은 두 개 이상의 Gem을 함께 사용하면서 발견한 Contract Gap, Adapter, Validation, Recovery Policy 같은 새로운 지식을 담습니다.

```text
Origin A + Origin B
        ↓
  Contract Gap
        ↓
     Crafting
        ↓
   Mixed Gem C
```

## 🛡️ Public-safe by Design

이 저장소의 Gem은 특정 프로젝트를 복원할 수 있는 원문 보관소가 아닙니다. 공개 가능한 Engineering Decision만 남기는 것을 원칙으로 합니다.

프로젝트명, 고객/회사명, 개인 식별정보, 사설 도메인, 내부 API URL, 로컬 절대경로, 실제 DB/Table/Resource 이름, Secret/Token/Key, 프로젝트 고유 파일 구조는 Gem에서 제거하거나 일반화합니다.

자세한 기준은 [CONTRIBUTING.md](./CONTRIBUTING.md)를 참고해 주세요.

## 📚 Docs

- [Gem Programming](./docs/gem-programming.md)
- [Terminology](./docs/terminology.md)
- [Lifecycle](./docs/lifecycle.md)

## 🤝 Contributing

Gem Programming은 아직 초안입니다. Gem의 크기, metadata 형식, Origin/Mixed 관계, LLM이 안정적으로 구현하기 위해 필요한 정보 수준을 계속 실험하고 있습니다.

더 나은 구조나 추상화가 있다면 Issue로 의견을 남겨주세요. 새로운 Gem, 기존 Gem 개선, 문서 구조 변경에 대한 **PR도 환영합니다.**
