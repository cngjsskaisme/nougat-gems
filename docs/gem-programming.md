# 💎 Gem Programming

Gem Programming은 코드를 재사용의 최소 단위로 보지 않고, 실제 구현을 좋게 만든 **Engineering Decision**을 재사용 가능한 설계 단위로 다루는 실험적 방식입니다.

## Why

LLM은 같은 설계를 현재 기술 스택에 맞는 코드로 다시 구현할 수 있습니다. 따라서 오래 보존해야 할 자산을 특정 Framework의 코드에만 묶어둘 필요가 줄어듭니다.

```text
Project A
  ↓ Mine
Engineering Decisions
  ↓ Refine
Gem
  ↓ Socket with current context
Project B Implementation
```

## What makes a Gem

Gem은 Design Pattern보다 구체적이고 Code보다 추상적이어야 합니다.

다른 Agent가 원본 프로젝트를 보지 않아도 구현할 수 있도록 문제, 책임, Contract, Constraint, Data Flow, Failure Policy, Security Property, Validation Rule과 Trade-off를 충분히 남깁니다.

## Role of the Engineer

Engineer는 사용할 Gem, 현재 Socket, 허용되는 Constraint와 Validation Rule을 정합니다. LLM은 그 재료와 규칙 안에서 Context-specific Implementation을 생성하는 Crafting Engine에 가깝습니다.

## Non-determinism

같은 Gem에서도 구현 결과는 달라질 수 있습니다. 목표는 LLM을 결정론적으로 만드는 것이 아니라 **허용 가능한 결과의 경계**를 결정적으로 만드는 것입니다.

```text
LLM Output
   ↓
Contract
Schema
Security Rules
Tests
Validation
   ↓
Accepted Result
```
