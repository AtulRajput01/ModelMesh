# ModelMesh

**Enterprise AI Gateway with Intelligent Semantic Routing on AWS Bedrock**

ModelMesh is a production-ready AI gateway built on [LiteLLM](https://github.com/BerriAI/litellm) that exposes a single OpenAI-compatible inference endpoint while intelligently routing requests across multiple AWS Bedrock foundation models using semantic embeddings. It enforces AI governance through PII detection and prompt sanitization via Microsoft Presidio before any request reaches a model.

![ModelMesh Preview](assets/preview.png)

---

## Architecture

```
┌─────────────────────────────────────┐
│             API Client               │
│   (OpenAI SDK / curl / any HTTP)    │
└──────────────────┬──────────────────┘
                   │  POST /v1/chat/completions
                   ▼
┌─────────────────────────────────────┐
│         LiteLLM Gateway             │  ← Centralized auth, routing, logging
│         (port 4000)                 │
└──────┬───────────────────┬──────────┘
       │                   │
       ▼                   ▼
┌─────────────┐   ┌─────────────────┐
│  PostgreSQL │   │ Presidio Guards │  ← PII detection & anonymization
│  (metadata) │   │  Analyzer +     │     runs before every LLM call
└─────────────┘   │  Anonymizer     │
                  └────────┬────────┘
                           │  sanitized prompt
                           ▼
                  ┌─────────────────┐
                  │ Semantic Router │  ← Cohere multilingual embeddings
                  │  (router.json)  │     cosine similarity matching
                  └────────┬────────┘
                           │  selected model
                           ▼
                  ┌─────────────────────────────────┐
                  │         AWS Bedrock              │
                  │                                  │
                  │  ├── Claude Sonnet 4  (reasoning)│
                  │  ├── Gemma 3 27B     (writing)   │
                  │  ├── Llama 4 Maverick (creative) │
                  │  └── Nova Lite       (casual)    │
                  └─────────────────────────────────┘
```

---

## Features

### 1. Unified AI Gateway
A single OpenAI-compatible endpoint (`POST /v1/chat/completions`) abstracts all backend foundation models. Clients using the OpenAI SDK require zero code changes — only the `base_url` and API key need updating. Centralized authentication, request logging, and model configuration are managed through LiteLLM backed by PostgreSQL.

### 2. Semantic Model Routing
Requests are not routed by static rules. Each incoming prompt is embedded using **Cohere `embed-multilingual-v3`** via AWS Bedrock. The resulting vector is compared against a library of intent utterances defined in `router.json` using cosine similarity. The model whose utterances most closely match the request is automatically selected.

| Model | Inference Profile | Best For |
|---|---|---|
| Claude Sonnet 4 | `us.anthropic.claude-sonnet-4-6` | Complex reasoning, architecture, code review, security analysis |
| Gemma 3 27B | `google.gemma-3-27b-it` | Documentation, summarization, business writing, translation |
| Llama 4 Maverick | `meta.llama4-maverick-17b-instruct-v1:0` | Creative generation, marketing, ideation, blog writing |
| Nova Lite | `us.amazon.nova-lite-v1:0` | Lightweight conversations, casual queries, simple factual Q&A |

All models are accessed through **AWS Bedrock cross-region inference profiles**, enabling automatic failover and load distribution across AWS regions.

### 3. AI Guardrails — PII Detection & Prompt Sanitization
Every request passes through a **pre-call guardrail** powered by Microsoft Presidio before it reaches any foundation model. Personally identifiable information is detected and anonymized, enforcing data privacy policy at the infrastructure level rather than the application level.

Detected and anonymized entity types include:

- Email addresses
- Phone numbers
- Credit card numbers
- Aadhaar numbers
- Social Security Numbers (SSN)
- Personal names

### 4. Enterprise Benefits

- **Vendor abstraction** — swap or add models without changing client code
- **Cost optimization** — lightweight models handle simple tasks; premium models handle complex ones
- **Intelligent model selection** — semantic routing eliminates manual model selection
- **Centralized governance** — all policy enforcement, authentication, and logging in one layer
- **OpenAI SDK compatibility** — no client-side migration required
- **Cloud-native deployment** — fully containerized, orchestrated via Docker Compose

---

## Repository Structure

```
ModelMesh/
├── config.yml          # LiteLLM gateway config: master key, guardrail bindings, settings
├── docker-compose.yml  # Orchestrates LiteLLM, PostgreSQL, Presidio Analyzer, Presidio Anonymizer
├── router.json         # Semantic routing config: encoder, model routes, intent utterances
└── .env.example        # Environment variable template
```

**`config.yml`** — Configures the LiteLLM proxy: master key (loaded from env), database persistence, and the `presidio-pre-guard` guardrail binding that runs Presidio in `pre_call` mode.

**`docker-compose.yml`** — Defines four containerized services on a shared `litellm` Docker network: the LiteLLM gateway, PostgreSQL for model/config persistence, Presidio Analyzer (port 5002), and Presidio Anonymizer (port 5001).

**`router.json`** — Defines the semantic routing engine. Specifies `cohere.embed-multilingual-v3` as the encoder, then declares each model route with a name, set of representative utterances, and a similarity score threshold of `0.15`.

---

## Quick Start

### Prerequisites
- Docker and Docker Compose
- AWS account with Bedrock access enabled for the models listed above
- AWS credentials configured locally

### 1. Clone the repository
```bash
git clone https://github.com/your-username/ModelMesh.git
cd ModelMesh
```

### 2. Configure environment variables
```bash
cp .env.example .env
```

Edit `.env` with your values:
```env
LITELLM_MASTER_KEY=your-secure-master-key
POSTGRES_PASSWORD=your-postgres-password
DATABASE_URL=postgresql://litellm:your-postgres-password@postgres:5432/litellm
```

### 3. Configure AWS credentials
ModelMesh uses your local AWS credentials to call Bedrock. Ensure these are set:
```bash
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=us-east-1
```

Or use an IAM role if deploying on EC2/ECS.

### 4. Create the Docker network and start services
```bash
docker network create litellm
docker compose up -d
```

### 5. Verify services are running
```bash
docker compose ps
curl http://localhost:4000/health
```

---

## Example API Request

Send any prompt to the `smart-router` model — the semantic router selects the best foundation model automatically.

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-master-key" \
  -d '{
    "model": "smart-router",
    "messages": [
      {
        "role": "user",
        "content": "Design a high availability Kubernetes architecture for a payment processing system."
      }
    ]
  }'
```

This request will be routed to **Claude Sonnet 4** based on semantic similarity to architecture and system design utterances.

---

## Semantic Routing Flow

1. Client sends a `POST /v1/chat/completions` request with `model: smart-router`
2. LiteLLM receives the request and authenticates via master key
3. The prompt text is extracted and sent to the semantic router
4. The router calls `cohere.embed-multilingual-v3` on AWS Bedrock to produce a vector embedding of the prompt
5. Cosine similarity is computed against all pre-defined utterance embeddings in `router.json`
6. The route with the highest similarity score above the `0.15` threshold wins
7. LiteLLM rewrites the model target to the matched Bedrock inference profile
8. The request is forwarded to AWS Bedrock

---

## Guardrail Flow

1. An inbound request is intercepted by the `presidio-pre-guard` policy before the model call
2. The raw prompt is sent to **Presidio Analyzer** (`http://presidio-analyzer:3000`), which returns a list of detected PII entities with confidence scores and character spans
3. The prompt is forwarded to **Presidio Anonymizer** (`http://presidio-anonymizer:3000`) with the detected entity spans
4. Anonymizer replaces each entity with a typed placeholder (e.g., `<PERSON>`, `<EMAIL_ADDRESS>`, `<CREDIT_CARD>`)
5. The sanitized prompt continues to the routing engine and eventually to the foundation model
6. The original PII never leaves the gateway layer

---

## Technology Stack

| Layer | Technology | Role |
|---|---|---|
| Gateway | LiteLLM Proxy | OpenAI-compatible API, routing, auth, logging |
| Inference | AWS Bedrock | Managed foundation model inference |
| Routing | Semantic Router + Cohere Embeddings | Intent-based model selection |
| Guardrails | Microsoft Presidio | PII detection and prompt sanitization |
| Persistence | PostgreSQL 16 | Model config, keys, request metadata |
| Orchestration | Docker Compose | Containerized multi-service deployment |
| Configuration | YAML + JSON | Gateway and routing configuration |
| Language | Python (LiteLLM internals) | Runtime |

---

## Future Improvements

- **RAG Integration** — Retrieval-augmented generation with vector database backends (pgvector, Pinecone, Weaviate)
- **Redis Caching** — Semantic cache layer to deduplicate embeddings and reduce Bedrock API costs
- **Observability** — Prometheus metrics export and Grafana dashboards for request volume, latency, routing distribution, and cost per model
- **Kubernetes Deployment** — Helm chart for production-grade, horizontally scalable deployment
- **Multi-region** — Active-active Bedrock routing across AWS regions for latency optimization and resilience
- **Authentication Providers** — SSO/OIDC integration (Okta, Azure AD) for enterprise identity management
- **Rate Limiting** — Per-user and per-team token budgets with enforcement at the gateway layer
- **Cost Dashboards** — Real-time spend tracking and alerting per model, team, and use case
- **Additional Guardrails** — Prompt injection detection, toxicity filtering, and output validation

---

## License

MIT
