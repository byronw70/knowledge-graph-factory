# Knowledge-Graph Factory — Architecture Draft (v0.2)

Status: **draft for interview / technical decisions**. Where this document and the decision log disagree, the decision log wins (v1 scope has been narrowed: no vector index or property-graph store). Open questions are tracked in
[`DECISIONS.md`](DECISIONS.md).

## 1. Intent

A Docker-Compose, redeployable "knowledge-graph factory" that turns a provided source
bundle plus a build request (recipe, topic, language, required entities/relations/questions)
into a validated, versioned knowledge-graph release that can be queried, explored and
demonstrated — and torn down and recreated from that release.

Governing rule:

> **Models propose facts; validation and deterministic inference decide what is published.**

## 2. Logical architecture

```
 Source bundle + build request
            │
            ▼
 ┌────────────────────────┐      ┌──────────────────────────────┐
 │ control-api  (+ UI)    │─────▶│ metadata-db (jobs, releases) │
 └──────────┬─────────────┘      └──────────────────────────────┘
            │ enqueue
            ▼
 ┌────────────────────────┐ task calls  ┌───────────────────────────┐
 │ build-worker           │────────────▶│ inference-api (task layer)│
 │ parse → extract →      │             │ extract / match / embed / │
 │ resolve → map →        │             │ narrate, schema-checked   │
 │ validate → infer →     │             └────────────┬──────────────┘
 │ publish                │                          │ OpenAI-compatible
 └──────────┬─────────────┘                          ▼
            │                          ┌───────────────────────────┐
            │                          │ model-gateway             │
            │                          │ routing, keys, policy,    │
            │                          │ cache, cost/usage logging │
            │                          └──┬─────────┬──────────┬───┘
            │                             ▼         ▼          ▼
            │                        LM Studio  Anthropic   OpenAI
            │                        (local)    API         API
            ▼
 ┌──────────────────────────────┐
 │ Immutable release bundle      │  (object-store)
 │ asserted / inferred /         │
 │ validation / provenance       │
 └──────┬──────────────┬─────────┘
        ▼              ▼
  semantic-store   projections (disposable)
  Fuseki / TDB2    property graph, vector index
        └──────┬───────┘
               ▼
      query APIs / explorer / grounded Q&A
```

## 3. Inference: where it sits

Three kinds of "inference", kept separate:

| Kind | Purpose | Where | Authoritative? |
|---|---|---|---|
| Semantic | RDFS / OWL subset / rules over accepted facts | build-worker (Jena) | Yes, deterministic |
| Statistical | entity extraction, classification, relation proposals, entity matching, embeddings | inference-api → model-gateway | No — candidates with confidence |
| Generative | explanations, grounded answers over query results | inference-api → model-gateway, behind query APIs | No — removable |

LLMs are never on the write path directly. They return schema-constrained candidates that
go through resolution, SHACL validation and (optionally) human approval before publication.

## 4. Pluggable model providers (new in v0.2)

### 4.1 Requirement

Per deployment — and optionally per build and per task — choose between:

- **Local**: a model served by **LM Studio** via its OpenAI-compatible server
  (default `http://<host>:1234/v1`).
- **Anthropic**: Claude models with an API key from the Anthropic Console.
- **OpenAI**: GPT models with an OpenAI platform API key.

…and keep the option open for vLLM / llama.cpp / Ollama / a managed endpoint later without
code changes.

### 4.2 Two layers, deliberately

**`inference-api` (our code, thin).** Exposes *task* endpoints, not chat:

| Endpoint | Input | Output |
|---|---|---|
| `POST /tasks/extract` | text chunk + recipe extraction schema + language | candidate entities/relations, each with span, confidence |
| `POST /tasks/match` | candidate pair(s) + context | same/different/unsure + confidence + rationale |
| `POST /tasks/classify` | item + taxonomy | label(s) + confidence |
| `POST /tasks/embed` | texts | vectors + embedding-model id |
| `POST /tasks/narrate` | query results + evidence | grounded answer citing fact/evidence ids |

It owns prompts (versioned with the recipe), JSON-schema validation of every model response,
retry-on-invalid, chunking, and stamping provenance (`provider`, `model`, `prompt_version`,
`params`, `request_hash`) onto every candidate. The worker never knows which vendor answered.

**`model-gateway` (off the shelf).** One OpenAI-compatible endpoint that routes model aliases
to providers. Recommended: **LiteLLM Proxy** in a container — it already handles OpenAI,
Anthropic and any OpenAI-compatible base URL (LM Studio), key management, fallbacks, rate
limits, spend tracking and request logging. Pin the image by digest and scan it (LiteLLM
had a compromised PyPI release in 2026; treat it like any third-party dependency).
Fallback option if we'd rather not run a gateway: a ~200-line provider adapter inside
`inference-api` using the official `openai` and `anthropic` SDKs. The task-layer contract
is identical either way, so this is a reversible choice.

### 4.3 Model aliases, not model names

Recipes and code refer to **aliases**; deployment config maps aliases to providers.

```yaml
# config/models.yaml  (deployment-level; secrets come from env / Docker secrets)
profiles:
  local:            # everything on LM Studio — offline-capable
    extract:  { provider: lmstudio,  model: qwen3-14b-instruct }
    match:    { provider: lmstudio,  model: qwen3-14b-instruct }
    narrate:  { provider: lmstudio,  model: qwen3-14b-instruct }
    embed:    { provider: lmstudio,  model: nomic-embed-text-v1.5 }

  hybrid:           # local embeddings, cloud reasoning
    extract:  { provider: anthropic, model: <claude-model-id> }
    match:    { provider: anthropic, model: <claude-model-id> }
    narrate:  { provider: openai,    model: <gpt-model-id> }
    embed:    { provider: lmstudio,  model: nomic-embed-text-v1.5 }

  cloud:
    extract:  { provider: openai,    model: <gpt-model-id> }
    match:    { provider: openai,    model: <gpt-model-id> }
    narrate:  { provider: anthropic, model: <claude-model-id> }
    embed:    { provider: openai,    model: text-embedding-3-large }

providers:
  lmstudio:  { base_url: "http://host.docker.internal:1234/v1", api_key_env: LMSTUDIO_API_KEY }
  anthropic: { api_key_env: ANTHROPIC_API_KEY }
  openai:    { api_key_env: OPENAI_API_KEY }

defaults:
  temperature: 0
  max_retries_on_invalid_json: 2
```

A build request selects a profile (`model_profile: local`) and may override per task, subject
to policy (4.5).

### 4.4 Provider-specific facts the design has to absorb

- **LM Studio runs on the host, not in Compose.** Containers reach it via
  `host.docker.internal:1234` (on Linux add `extra_hosts: ["host.docker.internal:host-gateway"]`),
  and LM Studio's server must be set to listen on the network, not only localhost. A model
  must be loaded (or JIT-loading enabled). For headless/server deployments we can later swap
  LM Studio for a containerised llama.cpp or vLLM behind the same alias.
- **Structured output differs per vendor.** OpenAI and LM Studio accept
  `response_format: json_schema`; Anthropic uses its structured-output / tool-use mechanism.
  The gateway or adapter normalises this, and `inference-api` validates *every* response
  against the schema regardless — small local models will sometimes produce invalid JSON.
- **Anthropic has no embeddings endpoint.** Embeddings come from LM Studio (local embedding
  model), OpenAI, or a dedicated provider. The `embed` alias is therefore usually
  independent of the reasoning provider.
- **Embeddings are not interchangeable across models.** The embedding model id is recorded in
  the release manifest; changing it means re-embedding (it's a projection, so that's cheap
  and safe).
- **Cloud models are not bit-reproducible.** See 4.6.

### 4.5 Policy: who may see what data

Each build carries a sensitivity classification (e.g. `public | internal | restricted`).
The gateway/task layer enforces an allow-list:

```yaml
policy:
  public:     [lmstudio, anthropic, openai]
  internal:   [lmstudio, anthropic]        # e.g. only providers with an agreed DPA/zero-retention
  restricted: [lmstudio]                   # never leaves the machine
```

A build whose profile violates its classification is rejected at submission, not halfway
through. Air-gapped mode = `local` profile + no egress from the Compose network.

### 4.6 Reproducibility with non-deterministic models

- `temperature: 0`, fixed seeds where supported.
- **Response cache** keyed by `hash(provider, model, prompt_version, params, input)`; stored
  with the release's intermediates. Rebuilding from the same bundle replays cached model
  outputs, so the published graph is reproducible even when the provider isn't.
- Every candidate fact carries the provider/model/prompt version, so two releases built with
  different profiles can be diffed and compared (a good demo in itself: "local vs Claude vs
  GPT extraction quality on the same bundle").

### 4.7 Secrets

API keys only via `.env` (git-ignored) for local demos, Docker secrets / a vault for anything
shared. Keys never enter recipes, release bundles, logs or the metadata DB. The UI shows which
providers are *configured*, never the keys.

## 5. Services and Compose profiles

| Service | Responsibility | Profile |
|---|---|---|
| control-api | build submission, recipe selection, release catalogue, query routing | core |
| build-worker | parse, extract, resolve, map, validate, infer, publish | core |
| metadata-db | Postgres (or SQLite for single-user demos) | core |
| object-store | MinIO, or a mounted volume for the light profile | core |
| semantic-store | Apache Jena Fuseki / TDB2 | core |
| inference-api | task layer (4.2) | ai |
| model-gateway | LiteLLM Proxy (4.2) | ai |
| graph-store | Kuzu (embedded) or Neo4j projection | visual |
| explorer-web | upload/build UI, graph explorer, evidence viewer | visual |
| vector-index | Qdrant, only if hybrid retrieval is needed | search |

LM Studio is *external* to Compose (host app). A `local-llm` profile can add a containerised
llama.cpp/vLLM server for machines without LM Studio.

## 6. Recipes, releases, lifecycle

Unchanged from v0.1 (see session notes): versioned recipe packs
(`manifest.yaml, ontology.ttl, shapes.ttl, mappings/, rules/, extraction-schema.json,
prompts/, retrieval.yaml, views/, tests/`) and immutable release bundles
(`manifest.json, sources.json, asserted.ttl, inferred.ttl, validation-report.ttl,
provenance.ttl, projections, metrics.json, checksums.sha256`).

Additions for v0.2:

- `prompts/` in each recipe, versioned; prompt version stamped on every candidate.
- Release `manifest.json` records the model profile actually used per task and the
  embedding model id.
- `model-cache/` (optional, in intermediates) for reproducible rebuilds.

## 7. Delivery sequence (revised)

1. **Spike** — recipe + release contracts; Fuseki + SHACL + one rule; `inference-api` with
   `/tasks/extract` working against **LM Studio and one cloud provider** through the gateway;
   prove reproducible rebuild via the response cache.
2. **Minimal factory** — control-api, worker, file/CSV/JSON/document adapters, one recipe,
   release publishing, basic explorer, profile selection in the build request.
3. **Breadth** — remaining recipes, multilingual, property-graph projection, provider
   comparison report, hybrid search if needed.
4. **Hardening** — auth, policy enforcement, secrets, scanning, OpenTelemetry, backup/restore,
   evaluation sets per task and per provider.
