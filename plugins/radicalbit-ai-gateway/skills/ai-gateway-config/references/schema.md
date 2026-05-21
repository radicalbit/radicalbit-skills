# Radicalbit AI Gateway - Configuration Schema Reference

## Top-Level Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `chat_models` | list | Yes | Language models available to the gateway |
| `embedding_models` | list | No | Embedding models (required for semantic caching) |
| `guardrails` | list | No | Guardrail definitions (referenced by name in routes) |
| `routing` | list | No | Advanced routing rule definitions (referenced by name in routes) |
| `cache` | object | No | Redis backend config (required if any route uses caching) |
| `routes` | object | Yes | Named route definitions |

---

## `chat_models[]`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model_id` | string | Yes | Unique identifier used to reference this model in routes, fallbacks, and routing |
| `model` | string | Yes | Provider/model path — see provider formats below |
| `credentials.api_key` | string | Yes* | API key using `!secret ENV_VAR_NAME` syntax. *Not required for `mock` provider or self-hosted models (`base_url` set) — omit entirely for those |
| `credentials.base_url` | string | No | Base URL for self-hosted/compatible models (Ollama, vLLM, OpenRouter). Must end with `/v1` |
| `credentials.api_version` | string | No | Azure deployments only |
| `credentials.azure_ad_token` | string | No | Azure AD token, using `!secret ENV_VAR_NAME` syntax |
| `prompt` | string | No | Inline system prompt text. **Mutually exclusive with `prompt_ref`** |
| `prompt_ref` | string | No | Filename of an external `.md` prompt file (resolved via `PROMPTS_DIR` env var). **Mutually exclusive with `prompt`** |
| `role` | string | No | Context role when injecting a prompt: `system` or `developer` |
| `params.temperature` | float | No | Sampling temperature (0.0–1.0) |
| `params.max_tokens` | int | No | Maximum tokens in the response |
| `retry_attempts` | int | No | Number of retries on model failure |
| `input_cost_per_million_tokens` | float | No | Cost in USD per 1M input tokens (for cost tracking UI) |
| `output_cost_per_million_tokens` | float | No | Cost in USD per 1M output tokens (for cost tracking UI) |

### Provider Format

| Provider | `model` format | Notes |
|----------|----------------|-------|
| OpenAI | `openai/gpt-4o` | Standard |
| Anthropic | `anthropic/claude-3-5-sonnet-latest` | |
| Google Gemini (chat) | `google-genai/gemini-1.5-pro` | `api_key` required (no env fallback) |
| Ollama / vLLM / OpenRouter | `openai/model-name` | Add `base_url` to credentials |
| Azure OpenAI | `openai/model-name` | Add `api_version` + `azure_ad_token` |
| Mock (testing) | `mock/gateway` | No credentials needed |

### Mock provider params

| Field | Type | Description |
|-------|------|-------------|
| `params.latency_ms` | int | Simulated response latency |
| `params.response_text` | string | Fixed response content returned |

---

## `embedding_models[]`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model_id` | string | Yes | Unique identifier |
| `model` | string | Yes | Provider/model path — e.g. `openai/text-embedding-3-small`, `google-genai/models/gemini-embedding-001` |
| `credentials.api_key` | string | Yes* | API key. *Not required for `mock` provider or self-hosted models (`base_url` set). Required for Gemini (no env fallback) |
| `credentials.base_url` | string | No | Base URL for self-hosted models |
| `params.task_type` | string | No | Gemini only: `RETRIEVAL_QUERY`, `RETRIEVAL_DOCUMENT`, `SEMANTIC_SIMILARITY`, `CLASSIFICATION`, `CLUSTERING` |

### Mock embedding provider params

| Field | Type | Description |
|-------|------|-------------|
| `params.latency_ms` | int | Simulated latency |
| `params.vector_size` | int | Size of the returned embedding vector |

---

## `guardrails[]`

All guardrail types share these common fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier, referenced by routes |
| `type` | string | Yes | Guardrail type (see below) |
| `description` | string | No | Human-readable description |
| `where` | string | Yes | When to apply: `input`, `output`, or `io` (both) |
| `behavior` | string | No | Action on trigger: `block`, `soft_block`, `warn`. Not used by `presidio_anonymizer` (it always redacts) |
| `response_message` | string | No | Custom message returned to the user when triggered |

### Behavior Options

| Value | Effect |
|-------|--------|
| `block` | Hard reject — returns error |
| `soft_block` | Reject with user-friendly `response_message` |
| `warn` | Log a warning but allow the request to continue |

### Type: `starts_with`, `ends_with`, `contains`, `regex`

String pattern matching guardrails. **`parameters.values` is a list**, not a single string.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters.values` | list of strings | Yes | One or more patterns to match against |

Example:
```yaml
- name: block-competitors
  type: contains
  where: input
  behavior: soft_block
  parameters:
    values:
      - "competitor_a"
      - "competitor_b"
```

### Type: `presidio_analyzer`

Detects PII using Microsoft Presidio. Blocks or warns when PII is found.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters.language` | string | Yes | Language code: `en`, `it`, etc. |
| `parameters.entities` | list | Yes | List of entity types to detect |

### Type: `presidio_anonymizer`

Detects and **masks** PII. No `behavior` needed — it always redacts.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters.language` | string | Yes | Language code: `en`, `it`, etc. |
| `parameters.entities` | list | Yes | List of entity types to anonymize |

### Common Presidio Entity Types

`EMAIL_ADDRESS`, `PHONE_NUMBER`, `PERSON`, `LOCATION`, `DATE_TIME`, `SSN`, `IBAN_CODE`, `CREDIT_CARD`, `IT_IDENTITY_CARD`, `IP_ADDRESS`, `URL`, `NRP`

### Type: `judge`

Uses an LLM to semantically evaluate content.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters.prompt_ref` | string | Yes | Prompt template filename |
| `parameters.model_id` | string | Yes | `model_id` of the judge LLM |
| `parameters.temperature` | float | No | Sampling temperature for the judge |
| `parameters.max_tokens` | int | No | Max tokens for the judge response |
| `parameters.fallback_model_id` | string | No | Fallback judge model if primary is unavailable |

**Built-in prompt templates:**

| Template | Purpose |
|----------|---------|
| `toxicity_check.md` | Detects offensive or harmful content |
| `business_context_check.md` | Validates domain relevance |
| `prompt_injection_check.md` | Detects jailbreak / injection attempts |

Custom templates: set `JUDGE_PROMPTS_DIR` env var to your prompts directory.

---

## `routing[]`

Defined at top level, referenced inside a route via `routing: <name>`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier, referenced by routes |
| `type` | string | Yes | `deterministic` or `text_classification` |
| `default_model_id` | string | Yes | Fallback model when no rule matches or an error occurs |

### Type: `deterministic`

Evaluates rules locally with negligible overhead. Rule selected by `rule` field.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rule` | string | Yes | `keyword`, `token_length`, `time`, or `budget` |
| `output_mapping` | list | Yes | List of `{ model_id, conditions }` entries |

#### Rule: `keyword`

Scans human messages (case-insensitive) and selects the first mapping whose keyword list contains a match.

```yaml
output_mapping:
  - model_id: billing-model
    conditions:
      - billing
      - invoice
```

#### Rule: `token_length`

Routes based on token count of the **last user message**. Each entry's `conditions` must have exactly one of `gte`, `lte`, or `between`.

```yaml
output_mapping:
  - model_id: cheap-model
    conditions:
      lte: 999              # last message <= 999 tokens
  - model_id: long-context
    conditions:
      gte: 1000             # last message >= 1000 tokens
```

Use `between: [min, max]` for a bounded range (inclusive). Ranges must not overlap.

#### Rule: `context_length`

Same condition format as `token_length`, but routes based on the **total token count of the entire conversation** (all messages combined).

```yaml
output_mapping:
  - model_id: standard-model
    conditions:
      lte: 7999
  - model_id: long-context-model
    conditions:
      gte: 8000
```

#### Rule: `time`

Evaluates cron expressions against current UTC time. Selects the first matching entry.

```yaml
output_mapping:
  - model_id: business-hours-model
    conditions:
      - "* 9-17 * * 1-5"   # Mon–Fri 9am–5pm UTC
```

#### Rule: `budget`

Selects based on fraction of budget consumed (`0.0`–`1.0`). Requires `budget_limiting` on the same route.

```yaml
output_mapping:
  - model_id: mid-tier-model
    conditions:
      threshold: 0.6        # >= 60% consumed
  - model_id: cheap-model
    conditions:
      threshold: 0.8        # >= 80% consumed
```

### Type: `text_classification`

Delegates to an external ML classifier via HTTP.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | Yes | HTTP endpoint of the classifier |
| `timeout` | float | No | Request timeout in seconds (default: `5.0`) |
| `output_mapping` | list | Yes | Maps class labels to model IDs |

The gateway POSTs `{"dataframe_records": [{"inputs": "<last user message>"}]}` and expects `{"predictions": [{"class": "CLASS_A", "score": 0.95}]}`.

### Type: `semantic`

Routes by intent using embedding similarity. At startup, example utterances are embedded and averaged into one centroid vector per model. Each request embeds the last user message and routes to the model with the highest cosine similarity above the threshold — no external HTTP call per request.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `embedding_model_id` | string | Yes | References a top-level `embedding_models` entry |
| `similarity_threshold` | float | No | Minimum cosine similarity to match (default: `0.35`, range: 0.0–1.0) |
| `output_mapping` | list | Yes | Each entry: `model_id` + `conditions` (list of example utterances) |

`conditions` is a list of example utterances (5–10 per model recommended). Entry order does not affect selection — highest-scoring centroid wins.

```yaml
routing:
  - name: intent-routing
    type: semantic
    default_model_id: gpt-4o-mini
    embedding_model_id: text-embedding-3-small
    similarity_threshold: 0.35
    output_mapping:
      - model_id: code-model
        conditions:
          - "write a python function"
          - "debug this code"
          - "explain this algorithm"
      - model_id: general-model
        conditions:
          - "what is the weather"
          - "tell me a joke"
          - "summarize this article"
```

The embedding model must also be listed in the route's `embedding_models`.

---

## `cache`

Required at top level when any route uses caching.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `redis_host` | string | Yes | Redis/Valkey hostname |
| `redis_port` | int | Yes | Redis/Valkey port (typically `6379`) |

---

## `routes.<route-name>`

Route names should be **kebab-case** and descriptive (e.g., `customer-service`, `internal-qa`).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `chat_models` | list | Yes | List of `model_id` strings |
| `embedding_models` | list | No | List of embedding `model_id` strings (needed for semantic caching) |
| `guardrails` | list | No | List of guardrail names (must be defined globally) |
| `fallback` | list | No | Fallback chains — same structure as top-level, or can be defined here at route level |
| `routing` | string | No | Name of a `routing` entry defined at the top level |
| `caching` | object | No | Caching configuration |
| `rate_limiting` | object | No | Rate limiting configuration |
| `token_limiting` | object | No | Token limiting configuration |
| `budget_limiting` | object | No | Budget limiting (required for `budget` routing rule) |

### `routes.<name>.fallback[]`

> **Important:** Every `model_id` referenced in `target` or `fallbacks` **must also appear** in the route's `chat_models` list (or `embedding_models` for embedding fallbacks). Referencing a model not listed in the route causes a runtime error.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `target` | string | Yes | `model_id` of the primary model — must be in `chat_models` |
| `fallbacks` | list | Yes | Ordered list of fallback `model_id` values — each must be in `chat_models` |
| `type` | string | No | `embedding` for embedding fallbacks; omit for chat models |

### `routes.<name>.caching`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `exact` or `semantic` — required |
| `enabled` | bool | No | `true` or `false` (defaults to true if section is present) |
| `ttl` | int | Yes | Cache TTL in seconds |
| `embedding_model_id` | string | Semantic only | Must match an `embedding_models` entry |
| `similarity_threshold` | float | Semantic only | Match threshold (0.0–1.0), e.g. `0.80` |
| `distance_metric` | string | Semantic only | `cosine` or `euclidean` |
| `dim` | int | Semantic only | Embedding vector dimensions (e.g. `1536` for `text-embedding-3-small`) |

### `routes.<name>.rate_limiting`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `algorithm` | string | Yes | `fixed_window` or `aligned_fixed_window` |
| `window_size` | string | Yes | e.g. `1 minute`, `30 seconds`, `1 hour` |
| `max_requests` | int | Yes | Maximum number of requests in the window |

### `routes.<name>.token_limiting`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `algorithm` | string | No | `fixed_window` or `aligned_fixed_window` |
| `input.window_size` | string | No | e.g. `10 seconds`, `1 minute` |
| `input.max_token` | int | No | Max input tokens per window |
| `output.window_size` | string | No | e.g. `10 minutes` |
| `output.max_token` | int | No | Max output tokens per window |

### `routes.<name>.budget_limiting`

Required when using the `budget` routing rule.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `input.algorithm` | string | Yes | `fixed_window` or `aligned_fixed_window` |
| `input.window_size` | string | Yes | e.g. `1 hour`, `1 day` |
| `input.max_budget` | float | Yes | Maximum USD budget per window |
