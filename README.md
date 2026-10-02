# knowledge-graph-factory

A redeployable, Docker Compose-based factory for building validated, versioned knowledge graphs
from provided data (documents, tabular data, existing ontologies), driven by reusable graph recipes.

Model inference is pluggable: run locally against LM Studio's OpenAI-compatible server, or use
frontier models via Anthropic or OpenAI API keys — selectable per deployment, build and task,
subject to a per-build data-sensitivity policy.

Guiding rule: **models propose facts; validation and deterministic inference decide what is published.**

## Status

Design phase — no code yet.

- [Architecture draft](docs/ARCHITECTURE.md)
- [Decision log / open questions](docs/DECISIONS.md)
