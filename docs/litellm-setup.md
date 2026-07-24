# LiteLLM Setup

Unified LLM API proxy that exposes different models of various providers through a single OpenAI-compatible endpoint at [llm.lukaswoellhaf.com](https://llm.lukaswoellhaf.com). Supports both [Open WebUI](https://chat.lukaswoellhaf.com) and VS Code Copilot BYO-model usage.

## Architecture

```
    Open WebUI / VS Code Copilot
                    │
                    ▼
    llm.lukaswoellhaf.com (Traefik ingress)
                    │
                    ▼
    LiteLLM Proxy
    ┌───────────────────────────────────────┐
    │  Provider wildcards:                  │
    │  openai/*   → OpenAI API              │
    │  mistral/*  → Mistral API             │
    │  deepseek/* → DeepSeek API            │
    │  kimi-k3    → Moonshot API            │
    │  kimi-k2.6  → Moonshot API            │
    │                                       │
    │  Global settings:                     │
    │  • route_all_chat_openai_to_responses │
    │  • drop_params: true                  │
    │  • Pre-call hook for Kimi             │
    │  • Prometheus metrics                 │
    └───────────────────────────────────────┘
```

## Supported Models

LiteLLM uses provider wildcards (`openai/*`, `mistral/*`, `deepseek/*`) — any model supported by the upstream provider is automatically available without config changes. Two Kimi models are explicitly listed to include model metadata (token limits, vision support).

## Key Design Decisions

### Responses API Routing (`route_all_chat_openai_to_responses`)

All OpenAI `/chat/completions` traffic is auto-routed through OpenAI's `/v1/responses` API. This fixes a fundamental incompatibility: GPT-5.6 models (terra, luna, sol) reject `reasoning_effort` when tools (function calling) are present on the Chat Completions endpoint. The Responses API supports both natively.

```yaml
litellm_settings:
  route_all_chat_openai_to_responses: true
```

### Parameter Dropping (`drop_params`)

Unknown or incompatible request parameters are silently dropped instead of causing 400 errors. Critical for VS Code Copilot which sends model-specific params (`reasoning_effort`, `temperature`, etc.) that not all providers support.

```yaml
litellm_settings:
  drop_params: true
```

### Kimi Pre-Call Hook

Kimi K3 and K2.6 have **fixed** parameter values — any deviation returns a 400 error:

| Parameter | Fixed Value |
|---|---|
| `temperature` | 1.0 |
| `top_p` | 0.95 |
| `n` | 1 |
| `presence_penalty` | 0 |
| `frequency_penalty` | 0 |

LiteLLM's `litellm_params` can only set **defaults** — they don't override request-level values from clients. A pre-call hook intercepts all requests to moonshot models and forces these parameters regardless of what the client sends.

The hook lives in `applications/litellm/templates/hooks-configmap.yaml` as a LiteLLM `CustomLogger`:

```python
class ForceMoonshotParams(CustomLogger):
    async def async_pre_call_hook(self, user_api_key_dict, cache, data, call_type):
        model = str(data.get("model", ""))
        if "moonshot" in model or model in ("kimi-k3", "kimi-k2.6"):
            data["temperature"] = 1.0
            data["top_p"] = 0.95
            data["n"] = 1
            data["presence_penalty"] = 0
            data["frequency_penalty"] = 0
        return data
```

## VS Code Copilot BYO-Model

LiteLLM is configured as a custom provider in VS Code's `chatLanguageModels.json` (`~/Library/Application Support/Code/User/chatLanguageModels.json`). This lets Copilot use any model routed through LiteLLM as an alternative to the built-in Copilot models.

### Example Configuration

```json
{
  "name": "LiteLLM",
  "vendor": "customendpoint",
  "apiKey": "${input:chat.lm.secret.-20436fbf}",
  "apiType": "chat-completions",
  "models": [
    {
      "id": "openai/gpt-5.3-codex",
      "name": "GPT-5.3 Codex",
      "url": "https://llm.lukaswoellhaf.com/v1/chat/completions",
      "toolCalling": true,
      "vision": true,
      "maxInputTokens": 400000,
      "maxOutputTokens": 128000,
      "supportsReasoningEffort": ["low", "medium", "high", "xhigh"]
    }
  ],
  "settings": {
    "deepseek/deepseek-v4-pro": {
      "reasoningEffort": "high"
    }
  }
}
```

### API Key Management

The `apiKey` field uses `${input:chat.lm.secret.-20436fbf}` — VS Code prompts for the LiteLLM master key on first use and stores it in the system keychain.

## Monitoring

### Prometheus Metrics

LiteLLM exposes metrics at `/metrics` which are scraped by Prometheus every 60s via a `ServiceMonitor`:

```yaml
serviceMonitor:
  enabled: true
  interval: 60s
```

Metrics include request counts, latencies, token usage, and error rates per model/provider.

### Grafana Dashboard

A dedicated custom LiteLLM dashboard (`applications/monitoring/dashboards/litellm.json`) visualizes:
- Request volume and latency by model
- Token consumption and cost estimates
- Error rates and status codes
- Provider availability
