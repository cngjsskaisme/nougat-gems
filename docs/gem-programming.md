# 💎 Gem Programming

Gem Programming is an experimental approach that treats **engineering decisions**, rather than code, as the primary unit of reuse.

## Why

LLMs can reimplement the same design for the technology stack of a new project. That reduces the need to preserve reusable knowledge only as code tied to a specific framework.

```text
Project A
  ↓ Mine
Engineering Decisions
  ↓ Refine
Gem
  ↓ Socket with current context
Project B Implementation
```

## What Makes a Gem

A Gem should be more concrete than a design pattern but more abstract than source code.

Another agent should be able to implement it without access to the original project. A useful Gem therefore preserves enough information about the problem, responsibilities, contracts, constraints, data flow, failure policies, security properties, validation rules, and trade-offs.

## Role of the Engineer

The engineer selects the Gem, identifies the current Socket, and defines the constraints and validation rules that must hold. The LLM acts more like a **crafting engine**, generating a context-specific implementation from those materials and rules.

## Non-determinism

The same Gem may produce different implementations. The goal is not to make the LLM deterministic; it is to make the **boundary of acceptable outcomes** deterministic.

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
