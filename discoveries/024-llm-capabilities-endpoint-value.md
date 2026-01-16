# Discovery: Would a dedicated `/api/llm/capabilities` endpoint be valuable for exposing model lists, default mappings, and provider-specific features?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, a dedicated `/api/llm/capabilities` endpoint would provide significant value**, but the implementation complexity varies based on scope. The current system has **no model discovery mechanism** - clients must hardcode provider names and rely on server-side "default" resolution without knowing what that resolves to. A capabilities endpoint would address three key gaps:

1. **Model Discovery**: Clients cannot discover which models are available on a given server
2. **Default Transparency**: The "default" → specific model mapping is opaque to clients
3. **Feature Awareness**: No way to query provider-specific capabilities (vision, tool calling, context windows)

However, the **implementation complexity is moderate** because:
- Model lists are not currently maintained in the codebase - defaults are hardcoded constants
- Provider capabilities (vision, tools, context windows) would need to be manually cataloged
- The system intentionally avoids model validation to prevent maintaining a model catalog

A **minimal viable implementation** exposing just provider availability and default mappings would be low-effort and immediately useful. A **full capabilities API** with model lists and feature matrices would require ongoing maintenance.

## Evidence

**Current state: No model information exposed anywhere**

The `/health` endpoint exposes provider availability but zero model information:

```rust
// crates/loom-server/src/health.rs:86-98
pub struct LlmProviderHealth {
    pub name: String,
    pub status: HealthStatus,
    pub mode: Option<String>,        // "api_key" or "oauth_pool"
    pub pool: Option<AnthropicPoolHealth>,
    pub latency_ms: Option<u64>,
    pub error: Option<String>,
    // NOTE: No model information at all
}
```

**Default models are hardcoded constants** (`crates/loom-server-llm-service/src/service.rs:66-74`):

```rust
const DEFAULT_ANTHROPIC_MODEL: &str = "claude-opus-4-20250514";
const DEFAULT_OPENAI_MODEL: &str = "gpt-4o";
const DEFAULT_VERTEX_MODEL: &str = "gemini-1.5-pro";
```

These can be overridden via environment variables but clients have no way to discover the actual values.

**CLI hardcodes provider selection** (`crates/loom-cli/src/main.rs:100-102, 317-320`):

```rust
/// LLM provider to use (anthropic or openai)
#[arg(short, long, env = "LOOM_LLM_PROVIDER", default_value = "anthropic")]
provider: String,

// ...

let llm_provider = match provider.to_lowercase().as_str() {
    "anthropic" => LlmProvider::Anthropic,
    "openai" => LlmProvider::OpenAi,
    other => anyhow::bail!("Unknown LLM provider: {other}. Use 'anthropic' or 'openai'"),
};
```

**No model validation at server level** (from discovery 006):
> "There is no server-side model validation. The LlmService does not check if a model exists before forwarding the request."

**Key Files:**
- `crates/loom-server/src/health.rs` — Current health check with LLM provider status (no models)
- `crates/loom-server-llm-service/src/service.rs` — LlmService with hardcoded default models
- `crates/loom-cli/src/main.rs` — CLI with hardcoded provider list
- `crates/loom-server/src/api.rs` — Router where new endpoint would be added

## Implementation Options

### Option A: Minimal Capabilities Endpoint (Recommended First Step)

Low effort, high value. Expose what's already known:

```rust
// New route: GET /api/llm/capabilities (public, no auth required)

#[derive(Serialize, ToSchema)]
pub struct LlmCapabilitiesResponse {
    pub providers: Vec<ProviderCapabilities>,
}

#[derive(Serialize, ToSchema)]
pub struct ProviderCapabilities {
    pub name: String,           // "anthropic", "openai", "vertex"
    pub available: bool,        // Is this provider configured?
    pub default_model: String,  // What "default" resolves to
}

// Implementation: ~50 lines, reads from LlmService
async fn get_llm_capabilities(State(state): State<AppState>) -> Json<LlmCapabilitiesResponse> {
    let providers = match &state.llm_service {
        Some(service) => vec![
            ProviderCapabilities {
                name: "anthropic".into(),
                available: service.has_anthropic(),
                default_model: service.anthropic_default_model().into(),
            },
            // ... openai, vertex
        ],
        None => vec![],
    };
    Json(LlmCapabilitiesResponse { providers })
}
```

### Option B: Extended Capabilities with Model Lists

Higher effort, requires maintenance. Add supported model lists:

```rust
pub struct ProviderCapabilities {
    pub name: String,
    pub available: bool,
    pub default_model: String,
    pub supported_models: Vec<ModelInfo>,  // NEW
}

pub struct ModelInfo {
    pub id: String,                    // "claude-opus-4-20250514"
    pub display_name: String,          // "Claude Opus 4"
    pub context_window: u32,           // 200000
    pub supports_vision: bool,         // true
    pub supports_tools: bool,          // true
    pub max_output_tokens: Option<u32>, // 4096
}
```

**Challenge**: Model lists change frequently. Options:
1. Hardcode and update with releases (maintenance burden)
2. Fetch from provider APIs at startup (adds latency, requires API calls)
3. Make it configurable via server config (operator responsibility)

### Option C: Dynamic Model Discovery via Provider APIs

Highest effort, most accurate. Query providers for available models:

```rust
// Anthropic: No public model list API
// OpenAI: GET /v1/models returns available models
// Vertex: Requires GCP API calls

// Would need caching to avoid per-request API calls
```

**Not recommended** due to:
- Anthropic has no model list API
- Adds external dependencies to health-critical endpoint
- Caching complexity

## Implications

1. **Option A is the clear first step**: Minimal effort, solves the immediate problem of clients not knowing what's available. Can be implemented in <1 hour.

2. **Model lists are a maintenance burden**: If implemented, they'll need updating with each provider release. Consider making this opt-in or operator-configured.

3. **CLI could use this for better UX**: Instead of failing with "Unknown LLM provider", the CLI could query capabilities and show available options.

4. **Web UI benefits**: A capabilities endpoint enables dynamic provider/model selection in the web frontend.

5. **Backward compatibility**: This is purely additive - existing clients continue to work.

## Follow-up Questions

- [ ] What is the expected user experience during a retry wait? Should the CLI show a spinner or countdown?
- [ ] Should the capabilities endpoint require authentication, or be public like `/health`?

## Related Discoveries

- [[014-provider-model-discovery]] — Documents that `/health` is the current de facto discovery mechanism
- [[006-llm-service-model-resolution]] — How the server resolves "default" to specific model versions
- [[023-cli-health-check-on-startup]] — Proposed CLI health check that could use capabilities endpoint
