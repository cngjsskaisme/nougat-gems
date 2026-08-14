# 🧬 Mixed 001-002 — NDJSON × json-render Adapter

[Origin 001 Streaming Proxy](../../origin/001-streaming-proxy/)와 [Origin 002 LLM UI](../../origin/002-llm-ui/)를 함께 사용할 때 생기는 **Transport Event와 UI Spec lifecycle의 Contract Gap**을 연결하는 Mixed Gem입니다.

핵심 결과는 `Part Lifecycle`, `Working Spec / Render Spec`, `Last-Known-Good`, `Final Snapshot Reconciliation`입니다.

- [Backend](./backend.md)
- [Frontend](./frontend.md)
- [Metadata](./metadata.yaml)
