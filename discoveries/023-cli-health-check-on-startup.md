# Discovery: Should the CLI query `/health` on startup to validate provider availability before attempting LLM requests?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, but with important caveats about implementation approach.** Adding a startup health check would provide significant UX improvements by failing fast with actionable error messages, but it must be implemented carefully to avoid adding latency to the happy path and to handle edge cases gracefully.

The current CLI behavior is **reactive**: it attempts an LLM request and only discovers provider unavailability when the server returns a 503 error with a raw JSON body. A **proactive** health check would:

1. Validate server connectivity before starting the REPL
2. Confirm the requested provider (Anthropic/OpenAI) is configured
3. Provide clear, actionable error messages before the user invests time in composing a prompt
4. Potentially display degraded status warnings (e.g., "Anthropic pool has 1/3 accounts available")

However, there are trade-offs:
- **Latency**: The `/health` endpoint performs 11+ parallel checks (database, K8s, SMTP, etc.) and can take 100-500ms
- **False negatives**: A healthy provider at startup could become unavailable mid-session
- **Offline mode**: Some CLI commands (list, search with local fallback) don't require server connectivity

## Evidence

**Current startup flow** (`crates/loom-cli/src/main.rs:707-769`):

```rust
async fn start_repl_session(
    config: &loom_cli_config::LoomConfig,
    args: &Args,
    thread_store: Arc<dyn ThreadStore>,
    mut thread: Thread,
) -> Result<()> {
    // ... workspace setup ...
    
    let token = auth::load_token(&args.server_url).await;
    let llm_client = create_llm_client(&args.server_url, &args.provider, token.clone())?;
    
    // No health check here - goes straight to REPL
    run_repl(llm_client.as_ref(), /* ... */).await
}
```

**Health endpoint response includes LLM provider status** (`crates/loom-server/src/health.rs:352-450`):

```rust
pub async fn check_llm_providers(llm_service: Option<&LlmService>) -> LlmProvidersHealth {
    match llm_service {
        Some(service) => {
            let mut providers = Vec::new();
            
            if service.has_anthropic() {
                // Returns status, mode (api_key/oauth_pool), pool details
                providers.push(LlmProviderHealth { name: "anthropic", status, mode, pool, ... });
            }
            
            if service.has_openai() {
                providers.push(LlmProviderHealth { name: "openai", status: HealthStatus::Healthy, ... });
            }
            
            LlmProvidersHealth { status: overall_status, providers }
        }
        None => LlmProvidersHealth {
            status: HealthStatus::Degraded,
            providers: vec![LlmProviderHealth { name: "none", error: Some("LLM service not configured") }],
        },
    }
}
```

**Health response structure** (from live server):

```json
{
  "status": "healthy",
  "duration_ms": 127,
  "components": {
    "llm_providers": {
      "status": "healthy",
      "providers": [
        {"name": "anthropic", "status": "healthy", "mode": "oauth_pool", "pool": {...}},
        {"name": "openai", "status": "healthy"}
      ]
    }
  }
}
```

**Current error experience** (without health check):

```
$ loom
> What is 2+2?
Error: API error: proxy returned status 503 Service Unavailable: {"error":"service_unavailable","message":"Anthropic provider is not configured on the server"}
```

**Key Files:**
- `crates/loom-cli/src/main.rs` — CLI entry point where health check would be added
- `crates/loom-server/src/health.rs` — Health check implementation with LLM provider status
- `crates/loom-server/src/routes/health.rs` — `/health` endpoint handler
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient that could gain a `check_health()` method

## Implementation Recommendation

### Option A: Lightweight Provider-Only Check (Recommended)

Add a new endpoint `/api/llm/status` that returns only LLM provider availability:

```rust
// Server-side: minimal latency, no external service checks
#[get("/api/llm/status")]
async fn llm_status(State(state): State<AppState>) -> Json<LlmStatusResponse> {
    Json(LlmStatusResponse {
        anthropic: state.llm_service.as_ref().map(|s| s.has_anthropic()).unwrap_or(false),
        openai: state.llm_service.as_ref().map(|s| s.has_openai()).unwrap_or(false),
        vertex: state.llm_service.as_ref().map(|s| s.has_vertex()).unwrap_or(false),
    })
}

// CLI-side: fast check before REPL
async fn validate_provider(server_url: &str, provider: &str) -> Result<()> {
    let status = fetch_llm_status(server_url).await?;
    match provider {
        "anthropic" if !status.anthropic => bail!("Anthropic is not configured on {}. Contact your administrator.", server_url),
        "openai" if !status.openai => bail!("OpenAI is not configured on {}. Contact your administrator.", server_url),
        _ => Ok(())
    }
}
```

### Option B: Parse Existing `/health` Endpoint

Use the existing `/health` endpoint but only parse the `llm_providers` section:

```rust
async fn check_provider_health(server_url: &str, provider: &str) -> Result<ProviderStatus> {
    let health: HealthResponse = reqwest::get(format!("{}/health", server_url)).await?.json().await?;
    
    let provider_health = health.components.llm_providers.providers
        .iter()
        .find(|p| p.name == provider);
    
    match provider_health {
        Some(p) if p.status == HealthStatus::Healthy => Ok(ProviderStatus::Healthy),
        Some(p) if p.status == HealthStatus::Degraded => Ok(ProviderStatus::Degraded(p.error.clone())),
        Some(p) => Err(anyhow!("Provider {} is unhealthy: {:?}", provider, p.error)),
        None => Err(anyhow!("Provider {} is not configured on the server", provider)),
    }
}
```

### Option C: Lazy Validation with Caching

Don't check on startup, but cache the first error and provide better messaging:

```rust
// In ProxyLlmClient
fn handle_503_error(body: &str) -> LlmError {
    if let Ok(err) = serde_json::from_str::<ErrorResponse>(body) {
        if err.message.contains("not configured") {
            return LlmError::ProviderNotConfigured {
                provider: extract_provider(&err.message),
                message: err.message,
            };
        }
    }
    LlmError::Api(format!("503: {}", body))
}
```

## Implications

1. **UX Improvement**: Users would see clear, actionable errors before typing their first prompt, not after waiting for a response.

2. **Latency Trade-off**: Option A adds ~10-50ms, Option B adds ~100-500ms. Option C adds no startup latency.

3. **Graceful Degradation**: The check should be non-blocking for commands that don't need LLM (list, search, logout).

4. **Warning vs Error**: A degraded provider (e.g., OAuth pool with some accounts cooling) could show a warning but still allow the session to proceed.

5. **Offline Resilience**: If the health check fails due to network issues, the CLI should still attempt the LLM request (the server might be reachable for LLM but not for health).

## Follow-up Questions

- [ ] Would a dedicated `/api/llm/capabilities` endpoint be valuable for exposing model lists, default mappings, and provider-specific features?
- [ ] What is the expected user experience during a retry wait? Should the CLI show a spinner or countdown?

## Related Discoveries

- [[014-provider-model-discovery]] — Documents that `/health` is the current de facto discovery mechanism
- [[013-proxy-llm-client-503-handling]] — How the CLI currently handles 503 "provider not configured" errors
- [[022-llmerror-provider-not-configured-variant]] — Proposal for typed error variant for unconfigured providers
