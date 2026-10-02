# Decision log / interview tracker

## Context (from interview, 2026-10-02)

- Purpose: internal capability / R&D (not client-facing yet).
- Runs on: Apple Silicon laptop, Docker Desktop, LM Studio on the host.
- Data sensitivity: mixed — chosen per build.
- Scale: small (<100k facts per graph), English only for now.

| # | Question | Answer / leaning | Status |
|---|---|---|---|
| D1 | Graph families | Evidence & claims, entity/relationship, dependency/decision — one end-to-end first | **decided** (order open) |
| D2 | RDF-first vs property-graph-first | RDF-first; projections disposable | leaning |
| D3 | Inference permitted | RDFS + explicit rules | open |
| D4 | Model providers | Configurable local (LM Studio) or frontier (Anthropic/OpenAI) per deployment/build/task | **decided** |
| D5 | Gateway: LiteLLM Proxy vs in-house adapter | open | open |
| D6 | Hardware | Apple Silicon laptop | **decided** |
| D7 | Embedding source | Probably not needed in v1 (no vector search) | leaning |
| D8 | Sensitivity policy | Per-build classification; sensitive ⇒ local only | **decided** |
| D9 | Publishing | Auto-publish valid facts above threshold + human review queue | **decided** |
| D10 | Inputs | PDF/Word, CSV/Excel/JSON, existing ontologies/RDF | **decided** |
| D10b | Languages | English only, language-tagged for later | **decided** |
| D11 | Scale | Small (<100k facts) | **decided** |
| D12 | Offline / air-gapped | Supported via local profile; not a v1 requirement | leaning |
| D13 | Property-graph projection (Neo4j / Kuzu) | Probably defer; SPARQL + web graph view suffices at this scale | open |
| D14 | Decision-grade vs exploratory facts | open | open |
| D15 | Interrogation | Prompt → generated SPARQL → answer only from results, query + evidence shown | **decided** |
| D16 | Interface | One web app: upload → build → review → explore → ask | **decided** |
| D17 | Stack | Python backend, TypeScript/React web | **decided** |
| D18 | Repo | Private repo `byronw70/knowledge-graph-factory` | **decided** |
