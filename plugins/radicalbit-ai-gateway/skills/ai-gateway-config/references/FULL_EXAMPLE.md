# Radicalbit AI Gateway - Documentation

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Configuration (`config.yaml`)](#2-configuration-configyaml)
- [3. How to Use the Gateway](#3-how-to-use-the-gateway)
- [4. Deployment](#4-deployment)
- [5. Monitoring](#5-monitoring)
- [Appendix A: Mock Provider](#appendix-a-mock-provider)

---

## 1. Introduction

### What is the Radicalbit AI Gateway?

The Radicalbit AI Gateway is a powerful, flexible, and feature-rich service designed to act as a standardized, centralized, intelligent entry point for all your interactions with Large Language Models (LLMs). It provides a unified interface (standard OpenAI specification) to multiple AI models, abstracting away the complexity of dealing with different providers and adding a layer of control, security, and resilience to your AI applications.

#### A Non-Technical Overview

Think of the AI Gateway as a smart traffic controller for AI requests. When your application needs to talk to an AI model (like `gpt-4o` or `claude-sonnet-4`), instead of connecting directly, it goes through the gateway. The gateway then:

- **Routes requests intelligently** across multiple AI models to balance load
- **Provides backup options** if one model fails
- **Protects against abuse** by limiting how many requests can be made
- **Filters content** to prevent sensitive information from being processed
- **Caches responses** to improve performance and reduce costs
- **Monitors everything** to provide insights into usage patterns

### Key Features

- **OpenAI Compliant**: Completions endpoint is OpenAI compliant — compatible with OpenAI specification and libraries, no additional libraries needed.
- **Multiple Route Definition**: Create different "routes" within the same gateway, each with its own configuration for models, security, and traffic management.
- **Multiple Model Support**: Connect to OpenAI, Anthropic, and any OpenAI-compatible service like Ollama, vLLM, or OpenRouter.
- **Fallback Mechanisms**: Ensure high availability by automatically retrying failed requests with predefined backup models.
- **Guardrails**: Apply content filters and safety measures. Check for and block/warn about specific content, or redact sensitive information (PII) before it reaches a model.
- **Rate & Token Limiting**: Manage costs and prevent abuse by setting limits on requests or tokens over a time window.
- **Caching**: Reduce latency and costs by caching responses for repeated requests.
- **Observability**: Exposes detailed metrics via a Prometheus endpoint for easy integration with Grafana.

---

## 2. Configuration (`config.yaml`)

The gateway's entire behavior is controlled by a single YAML file. This file defines all routes, models, and the features applied to them.

### High-Level Structure

```yaml
# Definition of all routes (each route references top-level registries by ID)
routes:
  customer-service:
    chat_models: [ ... ]       # list[str] of chat model_ids
    embedding_models: [ ... ]  # list[str] of embedding model_ids
    fallback: [ ... ]          # fallback rules
    guardrails: [ ... ]        # list[str] of guardrail names
    rate_limiting: { ... }
    token_limiting: { ... }
    caching: { ... }

  another-route:
    # ... configuration for this route

# Top-level model registries (full model definitions)
chat_models:
  - model_id: ...
    # model configuration...
embedding_models:
  - model_id: ...
    # model configuration...

# Top-level guardrails registry (full guardrail definitions)
guardrails:
  - name: ...
    # guardrail configuration...

# Cache configuration (required if any route enables caching)
cache:
  redis_host: '...'
  redis_port: ...
```

### Configuration Breakdown

#### Routes

The top-level key is `routes`. Each key under `routes` defines a separate API endpoint with its own independent configuration.

#### Chat Models

The route's `chat_models` is a list of references to models defined in the top-level `chat_models` registry.

- `model_id`: Unique identifier used to reference this model in routes and fallbacks.
- `model`: The actual model identifier, formatted as `provider/model_name`.
  - **OpenAI:** `openai/model_name`
  - **OpenAI-Compatible (Ollama, vLLM, OpenRouter):** Prepend with `openai/`. Add `base_url` to credentials.
  - **Anthropic:** `anthropic/claude-3-5-sonnet-latest`
  - **Google Gemini:** `google-genai/model_name`. `api_key` is required (no environment variable fallback).
- `credentials`: Authentication details (`api_key`, `base_url` for self-hosted, `api_version`/`azure_ad_token` for Azure).
- `params`: Default parameters for every request (`temperature`, `max_tokens`, etc.).
- `prompt` / `prompt_ref`: Default system prompt — mutually exclusive.
- `role`: Role for prompt injection: `system` or `developer`.

#### Embedding Models

Same structure as `chat_models`. Required when any route uses semantic caching.

- For **Gemini embeddings**: set `params.task_type` (e.g., `RETRIEVAL_QUERY`, `SEMANTIC_SIMILARITY`).

#### Fallback Mechanisms (`fallback`)

Defines backup models if the primary fails. Gateway retries fallbacks in listed order.

```yaml
fallback:
  - target: gpt-4o
    fallbacks:
      - gpt-4o-mini
      - llama3-local
```

> Every `model_id` in `target` and `fallbacks` must also appear in the route's `chat_models` list.

#### Guardrails

Rules applied to `input`, `output`, or both (`io`) to enforce content policies.

**Guardrail types:**
- **`starts_with`, `ends_with`, `contains`, `regex`** — string pattern matching. `parameters.values` is always a list.
- **`presidio_analyzer`** — PII detection (block/warn).
- **`presidio_anonymizer`** — PII masking (always redacts, no `behavior` needed).
- **`judge`** — LLM-based semantic evaluation. Requires `parameters.prompt_ref` and `parameters.model_id`.

**Built-in judge templates:** `toxicity_check.md`, `business_context_check.md`, `prompt_injection_check.md`

#### Rate Limiting (`rate_limiting`)

```yaml
rate_limiting:
  algorithm: fixed_window
  window_size: 1 minute
  max_requests: 20
```

#### Token Limiting (`token_limiting`)

```yaml
token_limiting:
  input:
    window_size: 10 seconds
    max_token: 1000
  output:
    window_size: 10 minutes
    max_token: 500
```

#### Caching (`caching`)

```yaml
# In a route
caching:
  type: exact          # or semantic
  enabled: true
  ttl: 300

# At the top level (required when any route caches)
cache:
  redis_host: 'valkey'
  redis_port: 6379
```

For `semantic` caching, also set `embedding_model_id`, `similarity_threshold`, `distance_metric`, and `dim`.

#### Advanced Routing (`routing`)

Dynamically selects which model handles each request. Defined at top level, referenced by name in a route.

```yaml
routing:
  - name: my-routing-config
    type: deterministic | text_classification
    default_model_id: fallback-model

routes:
  my-route:
    chat_models: [model-a, model-b]
    routing: my-routing-config
```

**Deterministic rules:**

| Rule | Decision basis |
|------|---------------|
| `keyword` | Keywords in the user message |
| `token_length` | Input token count thresholds |
| `time` | Cron expressions against UTC time |
| `budget` | Fraction of budget consumed (requires `budget_limiting`) |

**Text classification routing:** Delegates to an external HTTP classifier. Gateway POSTs `{"dataframe_records": [{"inputs": "<message>"}]}` and expects `{"predictions": [{"class": "...", "score": 0.95}]}`.

---

## 3. How to Use the Gateway

The gateway exposes an OpenAI-compatible endpoint. Specify the **route name** in the `model` field of your request.

### API Endpoint

- **URL:** `/v1/chat/completions`
- **Method:** `POST`

### Example Request

```bash
curl http://127.0.0.1:9000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "customer-service",
    "messages": [
      {
        "role": "user",
        "content": "Hello, can you help me with my order?"
      }
    ]
  }'
```

---

## 4. Deployment

### Using Docker Compose

```bash
docker compose -f docker-compose-full.yaml --profile cache --profile metrics up
```

This starts the gateway, PostgreSQL, Valkey (Redis fork), Prometheus, and Grafana.

#### Mounting model prompts (`prompt_ref`)

Set environment variables and mount a host folder:

```yaml
environment:
  PROMPTS_DIR: "/radicalbit_ai_gateway/radicalbit_ai_gateway/prompts"
  JUDGE_PROMPTS_DIR: "/radicalbit_ai_gateway/radicalbit_ai_gateway/guardrails/judges/custom-prompts"
volumes:
  - ${PROMPTS_HOST_DIR:-./prompts}:/radicalbit_ai_gateway/radicalbit_ai_gateway/prompts:ro
```

---

## 5. Monitoring

### Prometheus Metrics

Metrics are exposed on port `8001`.

| Metric | Description |
|--------|-------------|
| `gateway_requests_total` | Total requests processed |
| `gateway_request_duration_milliseconds` | End-to-end request latency histogram |
| `gateway_model_invocations_total` | Model invocation count |
| `gateway_fallbacks_triggered_total` | Fallback trigger count |
| `gateway_guardrails_triggered_total` | Guardrail trigger count |
| `gateway_tokens_total_input_tokens_total` | Total input tokens processed |
| `gateway_tokens_total_output_tokens_total` | Total output tokens processed |
| `gateway_cache_hit_total` | Cache hit count |
| `gateway_rate_limiting_total` | Rate limit trigger count |

---

## Appendix A: Mock Provider

Test the gateway without real API calls or costs.

```yaml
chat_models:
  - model_id: mock-chat
    model: mock/gateway
    params:
      latency_ms: 150
      response_text: "mock response"
embedding_models:
  - model_id: mock-embed
    model: mock/embeddings
    params:
      latency_ms: 100
      vector_size: 8
routes:
  test-latency:
    chat_models:
      - mock-chat
    embedding_models:
      - mock-embed
```

No `credentials` required for `mock`. Generated embeddings are deterministic relative to the input string.
