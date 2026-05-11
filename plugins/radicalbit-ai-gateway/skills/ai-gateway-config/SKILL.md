---
name: ai-gateway-config
description: >
  Use this skill when the user asks to generate, create, configure, or update
  the Radicalbit AI Gateway configuration file (config.yaml). Triggers on phrases
  like "generate gateway config", "create config.yaml", "configure the gateway",
  "add a guardrail", "set up caching", "add rate limiting", "configure fallback",
  "add a model", "add a route", or any request about the AI Gateway YAML setup.
disable-model-invocation: true
---

# Radicalbit AI Gateway - Config Generator

Generate or update the `config.yaml` for the Radicalbit AI Gateway based on: $ARGUMENTS

## Your Task

1. **Understand the request** - determine whether the user wants to:
   - Generate a new `config.yaml` from scratch
   - Add or modify a specific section (model, route, guardrail, cache, etc.)
   - Review and improve an existing config

2. **If a `config.yaml` already exists in the current directory**, read it first before making changes.

3. **Gather missing information** - if the user's request is missing required fields, ask:
   - Which AI provider and model? (e.g., `openai/gpt-4o`, `anthropic/claude-3-5-sonnet`)
   - What should the route name be?
   - Are there any guardrails, caching, rate/token limits, or fallbacks needed?

4. **Use the reference files**:
   - [examples/example-config.yaml](examples/example-config.yaml) — structural skeleton with all sections
   - [references/schema.md](references/schema.md) — all valid fields, types, and constraints
   - [examples/full-config.yaml](examples/full-config.yaml) — a real-world complete example

5. **Write the output** to `config.yaml` in the current working directory.

6. **Summarize** what was generated: list routes, models, and features configured. Flag any fields that still need real values (e.g., API keys).

## Rules

- API keys must always use the `!secret ENV_VAR_NAME` syntax — never hardcode secrets.
- `prompt` and `prompt_ref` are mutually exclusive on a model — use one or the other.
- Guardrails are defined globally and referenced by name inside routes.
- String matching guardrails use `values: [...]` (a list), not a single `pattern` string.
- Caching requires `type: exact` or `type: semantic` — the field is mandatory.
- Embedding models are required when semantic caching is used on a route.
- Fallback can be defined at the top level or inside a route — both are valid.
- Every `model_id` referenced in `fallback` (both `target` and `fallbacks`) must also be listed in the route's `chat_models` (or `embedding_models` for embedding fallbacks). Missing this causes a runtime error.
- Route names should be kebab-case and descriptive (e.g., `customer-service`, `internal-qa`).
- For self-hosted or OpenAI-compatible models (Ollama, vLLM, OpenRouter), always add `base_url` to credentials and use `openai/` as the model prefix.
- Advanced routing (keyword, token_length, time, budget) is defined at top level under `routing` and referenced by name in routes.
- `budget_limiting` at route level is required when using the `budget` routing rule.
- The `mock` provider (`mock/gateway`, `mock/embeddings`) can be used for testing without real API calls.
