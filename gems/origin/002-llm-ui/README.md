# 💎 Origin 002 — LLM UI / json-render

A Gem for generating and validating UI Specs inside a constrained Catalog and Schema instead of allowing an LLM to emit arbitrary HTML or executable code.

## Intent

- restrict generatable Components, Props, and Actions through a Catalog
- separate LLM output from the actual UI implementation
- support progressive generation through SpecStream
- maintain validation boundaries on both Backend and Frontend

## Files

- [Backend](./backend.md)
- [Frontend](./frontend.md)
- [Metadata](./metadata.yaml)
