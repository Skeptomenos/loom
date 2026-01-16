# Discovery: Is there a mechanism for clients to discover which providers/models are available on a given server?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, but only through the `/health` endpoint, not a dedicated discovery API.** The server exposes LLM provider availability via the health check response, which includes a `llm_providers` component listing each configured provider with its status. However, there is **no dedicated endpoint for listing available models** or querying provider capabilities.

The `/health` endpoint returns an `LlmProvidersHealth` structure containing:
- Provider names (anthropic, openai, vertex)
- Health status (healthy, degraded, unhealthy)
- Mode (api_key or oauth_pool for Anthropic)
- Pool details for OAuth pool mode

Clients can parse this response to determine which providers are available, but they cannot discover:
1. Which specific models are supported by each provider
2. What the server's default model mappings are (e.g., "default" → "claude-opus-4-20250514")
3. Provider-specific capabilities or rate limits

The CLI currently hardcodes provider selection ("anthropic" or "openai") and relies on the server to return a 503 error with "provider not configured" message if the requested provider isn't available.

## Evidence

**Health endpoint response structure** (`crates/loom-server/src/health.rs:87-105`):

```rust
pub struct LlmProviderHealth {
    pub name: String,
    pub status: HealthStatus,
    pub mode: Option<String>,
    pub pool: Option<AnthropicPoolHealth>,
    pub latency_ms: Option<u64>,
    pub error: Option<String>,
}

pub struct LlmProvidersHealth {
    pub status: HealthStatus,
    pub providers: Vec<LlmProviderHealth>,
}
```

**Live health response example** (from `https://loom.ghuntley.com/health`):

```json
"llm_providers": {
  "status": "healthy",
  "providers": [
    {
      "name": "anthropic",
      "status": "healthy",
      "mode": "oauth_pool",
      "pool": {
        "accounts_total": 1,
        "accounts_available": 1,
        "accounts_cooling": 0,
        "accounts_disabled": 0,
        "accounts": [{"id": "claude-max-1767700740", "status": "available"}]
      }
    },
    {"name": "openai", "status": "healthy"}
  ]
}
```

**LlmService provider checking** (`crates/loom-server-llm-service/src/service.rs:265-278`):

```rust
/// Returns whether the Anthropic provider is configured.
pub fn has_anthropic(&self) -> bool {
    self.anthropic_client.is_some()
}

/// Returns whether the OpenAI provider is configured.
pub fn has_openai(&self) -> bool {
    self.openai_client.is_some()
}

/// Returns whether the Vertex AI provider is configured.
pub fn has_vertex(&self) -> bool {
    self.vertex_client.is_some()
}
```

**Default model mappings are server-side only** (`crates/loom-server-llm-service/src/service.rs:66-74`):

```rust
const DEFAULT_ANTHROPIC_MODEL: &str = "claude-opus-4-20250514";
const DEFAULT_OPENAI_MODEL: &str = "gpt-4o";
const DEFAULT_VERTEX_MODEL: &str = "gemini-1.5-pro";
```

**Key Files:**
- `crates/loom-server/src/health.rs` — Health check implementation with LLM provider status
- `crates/loom-server/src/routes/health.rs` — `/health` endpoint handler
- `crates/loom-server-llm-service/src/service.rs` — LlmService with provider availability methods
- `crates/loom-server/src/llm_proxy.rs` — Proxy handlers returning 503 for unconfigured providers
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient that doesn't query for availability

## Implications

1. **Clients must handle failures gracefully**: Without upfront discovery, clients learn about provider availability through 503 errors. The CLI should consider querying `/health` before attempting LLM requests.

2. **Model selection is opaque**: Clients cannot know what models are available or what "default" resolves to. This could lead to confusion when users specify models that don't exist on a particular provider.

3. **No capability negotiation**: There's no way for clients to discover provider-specific features (e.g., vision support, tool calling limits, context window sizes).

4. **Health endpoint is the de facto discovery mechanism**: Any client wanting to know provider availability should parse the `/health` response's `llm_providers.providers` array.

5. **Opportunity for improvement**: A dedicated `/api/llm/providers` or `/api/llm/models` endpoint could provide richer discovery information including supported models, default mappings, and capabilities.

## Follow-up Questions

- [ ] Should the CLI query `/health` on startup to validate provider availability before attempting LLM requests?
- [ ] Would a dedicated `/api/llm/capabilities` endpoint be valuable for exposing model lists, default mappings, and provider-specific features?

## Related Discoveries

- [[013-proxy-llm-client-503-handling]] — How the CLI handles 503 "provider not configured" errors
- [[006-llm-service-model-resolution]] — How the server resolves "default" to specific model versions
