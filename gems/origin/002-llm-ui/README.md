# 💎 Origin 002 — LLM UI / json-render

LLM이 임의 HTML이나 실행 코드를 생성하지 않고, 제한된 Catalog와 Schema 안에서 UI Spec을 만들고 검증하도록 하는 Gem입니다.

## Intent

- 생성 가능한 Component / Props / Action을 Catalog로 제한
- LLM Output과 실제 UI 구현 분리
- SpecStream을 통한 점진적 생성
- Backend와 Frontend 양쪽의 Validation Boundary 유지

## Files

- [Backend](./backend.md)
- [Frontend](./frontend.md)
- [Metadata](./metadata.yaml)
