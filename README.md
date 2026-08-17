# 💎 nougat-gems

> **Mine proven software. Refine the decisions. Socket the Gems. Regenerate the code.**

`nougat-gems` is an experimental **Gem Stash** for extracting proven **engineering decisions** from real-world software and preserving them as reusable design units called **Gems**.

Instead of copying old implementations, Gem Programming keeps the responsibilities, contracts, failure policies, security decisions, state transitions, and trade-offs that made those implementations work well—then lets an LLM or AI agent regenerate code for the context of a new project.

> 🚧 **Draft** — The Gem Programming format and repository structure are still evolving through real-world use. Issues and PRs proposing better structures, abstractions, or Gems are welcome.

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

One Gem lives in one directory. Even when a design spans different execution areas such as frontend and backend, documents stay under the same Gem when they share the same engineering intent.

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

A Gem is not code to copy. It is a **design to reimplement**.

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

The same Gem may produce very different code in NestJS, Go, Rust, or another environment. What matters is whether the implementation preserves the Gem's engineering intent and constraints.

## 🧬 Origin and Mixed Gems

An **Origin Gem** is a design independently extracted from a proven implementation.

A **Mixed Gem** captures new engineering knowledge discovered while combining two or more Gems—for example a contract gap, adapter, validation rule, or recovery policy.

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

This repository is not an archive of source projects. Gems should preserve only engineering decisions that are safe to publish.

Project names, customer/company names, personally identifying information, private domains, internal API URLs, absolute local paths, real database/table/resource names, secrets/tokens/keys, and project-specific file trees must be removed or generalized.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the detailed rules.

## 📚 Docs

- [Gem Programming](./docs/gem-programming.md)
- [Terminology](./docs/terminology.md)
- [Lifecycle](./docs/lifecycle.md)

## 🤝 Contributing

Gem Programming is still a draft. We are actively experimenting with Gem size, metadata format, Origin/Mixed relationships, and the amount of detail an LLM needs to produce reliable implementations.

If you see a better structure or abstraction, open an Issue. **PRs are welcome** for new Gems, improvements to existing Gems, and documentation changes.
