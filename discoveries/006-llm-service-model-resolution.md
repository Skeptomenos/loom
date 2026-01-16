# Discovery: How does the LlmService handle model resolution and what happens when a model is not available?

> Category: Follow-up (from Data Flow)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The LlmService handles model resolution through a **simple string substitution pattern**: when a client sends `"default"` as the model identifier, the server substitutes it with a provider-specific default model. This resolution happens **per-provider** at request time, just before forwarding to the actual LLM API. There is no complex model alias system or model availability validation - the server trusts that the resolved model exists at the provider.

### Model Resolution Flow

1. **Client sends request** with `model: "default"` (or any specific model name)
2. **LlmService receives request** via `complete_anthropic()`, `complete_openai()`, or `complete_vertex()`
3. **Resolution function called**: `resolve_anthropic_model()`, `resolve_openai_model()`, or `resolve_vertex_model()`
4. **Substitution logic**: If `request.model == "default"`, replace with the configured default; otherwise, pass through unchanged
5. **Forward to provider**: The resolved model name is sent to the actual LLM API

### Default Models (Hardcoded Constants)

| Provider | Default Model | Constant |
|----------|---------------|----------|
| Anthropic | `claude-opus-4-20250514` | `DEFAULT_ANTHROPIC_MODEL` |
| OpenAI | `gpt-4o` | `DEFAULT_OPENAI_MODEL` |
| Vertex | `gemini-1.5-pro` | `DEFAULT_VERTEX_MODEL` |

These defaults can be overridden via environment variables (`LOOM_SERVER_ANTHROPIC_MODEL`, etc.) at server startup.

### What Happens When a Model is Not Available

**There is no server-side model validation.** The LlmService does not check if a model exists before forwarding the request. If the client requests a non-existent model:

1. The request is forwarded to the provider API unchanged
2. The provider returns an error (e.g., `400 Bad Request` or `404 Not Found`)
3. The error is wrapped in `LlmError::Api` and returned to the client
4. The proxy converts this to `ServerError::UpstreamError` with HTTP 502

### What Happens When a Provider is Not Configured

Provider availability is checked **before** model resolution:

1. `has_anthropic()`, `has_openai()`, `has_vertex()` check if the client is configured
2. If not configured, return `ServerError::ServiceUnavailable` (HTTP 503) immediately
3. The error message is clear: "Anthropic provider is not configured on the server"

## Evidence

### Model Resolution Functions

```rust
// crates/loom-server-llm-service/src/service.rs:345-357
/// Substitute "default" model with the configured default for Anthropic.
fn resolve_anthropic_model(&self, request: LlmRequest) -> LlmRequest {
    if request.model == "default" {
        debug!(
            original_model = "default",
            resolved_model = %self.anthropic_model,
            "Substituting default model"
        );
        request.with_model(&self.anthropic_model)
    } else {
        request
    }
}
```

### Default Model Constants

```rust
// crates/loom-server-llm-service/src/service.rs:66-74
/// Default model for Anthropic when client sends "default".
/// Uses Opus 4 to match Claude Code's default for MAX subscribers.
const DEFAULT_ANTHROPIC_MODEL: &str = "claude-opus-4-20250514";

/// Default model for OpenAI when client sends "default".
const DEFAULT_OPENAI_MODEL: &str = "gpt-4o";

/// Default model for Vertex when client sends "default".
const DEFAULT_VERTEX_MODEL: &str = "gemini-1.5-pro";
```

### Default Override from Config

```rust
// crates/loom-server-llm-service/src/service.rs:224-235
let anthropic_model = config
    .anthropic_model
    .clone()
    .unwrap_or_else(|| DEFAULT_ANTHROPIC_MODEL.to_string());
let openai_model = config
    .openai_model
    .clone()
    .unwrap_or_else(|| DEFAULT_OPENAI_MODEL.to_string());
let vertex_model = config
    .vertex_model
    .clone()
    .unwrap_or_else(|| DEFAULT_VERTEX_MODEL.to_string());
```

### Provider Availability Check in Proxy

```rust
// crates/loom-server/src/llm_proxy.rs:141-146
if !service.has_anthropic() {
    tracing::error!("proxy_anthropic_stream: Anthropic provider not configured");
    return Err(ServerError::ServiceUnavailable(
        "Anthropic provider is not configured on the server".into(),
    ));
}
```

### Error Mapping for Invalid Models

```rust
// crates/loom-server/src/llm_proxy.rs:400-427
pub fn map_llm_error(err: LlmError) -> ServerError {
    match err {
        LlmError::Api(msg) => {
            tracing::warn!(error = %msg, "LLM API error");
            ServerError::UpstreamError(format!("LLM API error: {msg}"))
        }
        // ... other error types
    }
}
```

**Key Files:**
- `crates/loom-server-llm-service/src/service.rs` - LlmService with model resolution functions
- `crates/loom-server-llm-service/src/config.rs` - LlmServiceConfig with model override options
- `crates/loom-server/src/llm_proxy.rs` - Proxy handlers with provider availability checks
- `crates/loom-server-llm-service/src/error.rs` - Error types for service failures

## Implications

When working in this codebase:

- **No model validation at server level** - The server trusts the client's model choice. Invalid models will fail at the provider API level, not at the proxy level. This is intentional to avoid maintaining a model catalog.

- **"default" is a magic string** - Clients should use `"default"` to get the server's preferred model. This allows server operators to upgrade models without client changes.

- **Provider-specific defaults** - Each provider has its own default model. The Anthropic default (`claude-opus-4-20250514`) is notably the most expensive option, matching Claude Code's MAX subscriber default.

- **Environment variable overrides** - Server operators can change defaults via `LOOM_SERVER_ANTHROPIC_MODEL`, `LOOM_SERVER_OPENAI_MODEL`, `LOOM_SERVER_VERTEX_MODEL`.

- **Fail-fast on missing providers** - The proxy checks provider availability before attempting any LLM call, returning a clear 503 error. This is better UX than a cryptic 500 from a null pointer.

- **No model aliasing** - There's no system for mapping model aliases (e.g., "fast" → "claude-3-haiku"). Only "default" is special-cased.

## Follow-up Questions

- [ ] How does the CLI's ProxyLlmClient handle the 503 "provider not configured" error, and does it provide helpful user feedback?
- [ ] Is there a mechanism for clients to discover which providers/models are available on a given server?

## Related Discoveries

- [[003-request-flow]] - Documents the complete request flow including LlmService invocation
- [[005-post-tools-hook-auto-commit]] - Shows how auto-commit uses a hardcoded model (claude-3-haiku) rather than "default"
