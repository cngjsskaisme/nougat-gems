# 💎 Origin 001 — Streaming Proxy

A Gem for progressively delivering LLM output or other long-running results as an **NDJSON Event Stream**, while keeping Abort, Backpressure, and Retry boundaries explicit.

## Intent

- deliver Events as soon as they are ready instead of waiting for the entire response
- separate Transport from Business Logic
- propagate Client Disconnect into cancellation of internal work
- keep network Chunk boundaries separate from Event boundaries
- avoid unsafe full-request retries after partial output has already been consumed

## Files

- [Backend](./backend.md)
- [Frontend](./frontend.md)
- [Metadata](./metadata.yaml)
