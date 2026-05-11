# Radicalbit AI Gateway — Claude Code Plugin

Official [Claude Code](https://claude.ai/code) plugin from [Radicalbit](https://radicalbit.ai) for configuring and managing the Radicalbit AI Gateway.

## Skills

| Skill | Description |
|-------|-------------|
| [`/radicalbit-ai-gateway:ai-gateway-config`](plugins/radicalbit-ai-gateway/skills/ai-gateway-config/SKILL.md) | Generate or update the AI Gateway `config.yaml` |

## Installation

### Via Claude Code Marketplace

Add the marketplace and install the plugin:

```
/plugin marketplace add radicalbit/radicalbit-skills
/plugin install radicalbit-ai-gateway@radicalbit
```

Or browse interactively with `/plugin`.

### Team Installation (project-level)

Add to your project's `.claude/settings.json` to share with your team:

```json
{
  "extraKnownMarketplaces": {
    "radicalbit": {
      "source": {
        "source": "github",
        "repo": "radicalbit/radicalbit-skills"
      }
    }
  },
  "enabledPlugins": {
    "radicalbit-ai-gateway@radicalbit": true
  }
}
```

Team members receive installation prompts upon trusting the repository.

## Usage

Invoke skills using the `/radicalbit-ai-gateway:<skill-name>` syntax in Claude Code.

### `ai-gateway-config`

Generates or updates the Radicalbit AI Gateway `config.yaml` file based on your requirements.

**Trigger phrases:**
- "generate gateway config"
- "create config.yaml"
- "configure the gateway"
- "add a guardrail"
- "set up caching"
- "add rate limiting"
- "configure fallback"

**Example interaction:**

```
You: Generate a gateway config with two OpenAI routes — one for customer support
     with PII anonymization and semantic caching, and one internal QA route
     with keyword routing.

Claude: [Invokes /radicalbit-ai-gateway:ai-gateway-config and writes config.yaml]
```

**What the skill produces:**

- A complete `config.yaml` written to your current working directory
- Routes with models, guardrails, caching, rate/token limits, and fallbacks
- `!secret ENV_VAR_NAME` placeholders for all API keys (never hardcoded)
- A summary of what was generated and which fields need real values

## About the Radicalbit AI Gateway

The [Radicalbit AI Gateway](https://radicalbit.ai) is an OpenAI-compatible proxy that provides:

- **Multi-provider routing** — OpenAI, Anthropic, Ollama, vLLM, and more
- **Guardrails** — PII anonymization, toxicity detection, prompt injection blocking
- **Caching** — exact and semantic caching backed by Redis/Valkey
- **Rate & token limiting** — protect against abuse and control costs
- **Fallback chains** — automatic model failover for high availability
- **Advanced routing** — keyword, token-length, time-based, and budget-aware routing
- **Observability** — Prometheus metrics for Grafana dashboards

## License

Licensed under the [MIT License](LICENSE-MIT).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add or update skills.
