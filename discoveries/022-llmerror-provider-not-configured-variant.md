# Discovery: Would adding a `LlmError::ProviderNotConfigured` variant improve error handling and enable provider-specific guidance in the CLI?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, adding an `LlmError::ProviderNotConfigured` variant would significantly improve error handling and user experience.** The current implementation treats "provider not configured" errors as generic `LlmError::Api` errors, which prevents the CLI from:

1. **Distinguishing configuration errors from transient API failures** - Both are wrapped in `LlmError::Api`, making it impossible to provide targeted guidance
2. **Providing actionable user feedback** - Users see raw JSON error bodies instead of helpful messages like "Please configure the Anthropic API key on the server"
3. **Implementing provider-specific retry logic** - Configuration errors should not trigger retry attempts, unlike transient failures

The implementation would be straightforward:
- Add `ProviderNotConfigured { provider: String }` variant to `LlmError` in `loom-common-core`
- Update `ProxyLlmClient` to detect 503 responses with "provider not configured" messages
- Add i18n strings for provider-specific guidance (e.g., `client.error.provider_not_configured.anthropic`)

## Evidence

### Current LlmError Enum (No ProviderNotConfigured Variant)

```rust
// crates/loom-common-core/src/error.rs:31-48
#[derive(Clone, Error, Debug)]
pub enum LlmError {
    #[error("HTTP error: {0}")]
    Http(String),

    #[error("API error: {0}")]
    Api(String),

    #[error("Request timed out")]
    Timeout,

    #[error("Invalid response: {0}")]
    InvalidResponse(String),

    #[error("Rate limited: retry after {retry_after_secs:?} seconds")]
    RateLimited { retry_after_secs: Option<u64> },
}
```

### Server-Side: LlmServiceError Already Has ProviderNotConfigured

The server-side `LlmServiceError` already has a dedicated variant, demonstrating the pattern:

```rust
// crates/loom-server-llm-service/src/error.rs:9-19
#[derive(Debug, thiserror::Error)]
pub enum LlmServiceError {
    #[error("Configuration error: {0}")]
    Config(String),

    #[error("Provider not configured: {0}")]
    ProviderNotConfigured(String),  // <-- Already exists server-side

    #[error("LLM error: {0}")]
    Llm(#[from] LlmError),
}
```

### Current ProxyLlmClient Error Handling (Generic Fallback)

```rust
// crates/loom-server-llm-proxy/src/client.rs:138-151
if !status.is_success() {
    let error_body = response.text().await.unwrap_or_default();
    debug!(status = %status, body = %error_body, "proxy returned error status");

    if status == reqwest::StatusCode::TOO_MANY_REQUESTS {
        return Err(LlmError::RateLimited {
            retry_after_secs: None,
        });
    }

    // All other errors, including 503 "provider not configured", become generic Api errors
    return Err(LlmError::Api(format!(
        "proxy returned status {status}: {error_body}"
    )));
}
```

### Server Returns Structured JSON for 503 Errors

```rust
// crates/loom-server/src/error.rs:184-195
ServerError::ServiceUnavailable(msg) => {
    (
        StatusCode::SERVICE_UNAVAILABLE,
        ErrorResponse {
            error: "service_unavailable".to_string(),
            message: msg.clone(),  // e.g., "Anthropic provider is not configured on the server"
            server_version: None,
            client_version: None,
        },
    )
}
```

### Proposed Implementation

**Step 1: Add variant to LlmError**

```rust
// crates/loom-common-core/src/error.rs
#[derive(Clone, Error, Debug)]
pub enum LlmError {
    // ... existing variants ...

    #[error("Provider not configured: {provider}")]
    ProviderNotConfigured { provider: String },
}
```

**Step 2: Update ProxyLlmClient to detect and parse 503 errors**

```rust
// crates/loom-server-llm-proxy/src/client.rs

#[derive(Deserialize)]
struct ProxyErrorResponse {
    error: String,
    message: String,
}

// In error handling block:
if status == reqwest::StatusCode::SERVICE_UNAVAILABLE {
    if let Ok(err_response) = serde_json::from_str::<ProxyErrorResponse>(&error_body) {
        if err_response.message.contains("not configured") {
            // Extract provider name from message or use the client's configured provider
            return Err(LlmError::ProviderNotConfigured {
                provider: self.provider.to_string(),
            });
        }
    }
}
```

**Step 3: Add i18n strings for user-friendly messages**

```po
# crates/loom-common-i18n/locales/en/messages.po
msgid "client.error.provider_not_configured"
msgstr "The {provider} provider is not configured on the server. Please contact your administrator to set up the API key."

msgid "client.error.provider_not_configured.anthropic"
msgstr "Anthropic is not configured. The server administrator needs to set LOOM_SERVER_ANTHROPIC_API_KEY."

msgid "client.error.provider_not_configured.openai"
msgstr "OpenAI is not configured. The server administrator needs to set LOOM_SERVER_OPENAI_API_KEY."
```

**Step 4: Update CLI error display**

```rust
// crates/loom-cli/src/main.rs
Err(AgentError::Llm(LlmError::ProviderNotConfigured { provider })) => {
    let msg = loom_common_i18n::t_fmt(
        get_locale(),
        "client.error.provider_not_configured",
        &[("provider", &provider)],
    );
    eprintln!("{}", msg);
}
```

**Key Files:**
- `crates/loom-common-core/src/error.rs` — LlmError enum (add variant)
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient (detect 503 and parse)
- `crates/loom-common-i18n/locales/*/messages.po` — i18n strings (add provider-specific messages)
- `crates/loom-cli/src/main.rs` — CLI error display (handle new variant)

## Implications

1. **User Experience**: Users will see actionable error messages instead of raw JSON. A developer encountering "Anthropic is not configured" will know exactly what to do (or who to contact).

2. **Error Classification**: The CLI can now distinguish between:
   - **Configuration errors** (`ProviderNotConfigured`) — Don't retry, show guidance
   - **Rate limits** (`RateLimited`) — Retry after delay
   - **Transient failures** (`Http`, `Timeout`) — Retry immediately
   - **Permanent API errors** (`Api`) — Don't retry, investigate

3. **Retry Logic**: The Agent state machine can skip retry attempts for `ProviderNotConfigured` errors, avoiding wasted time and API calls.

4. **Extensibility**: The pattern can be extended for other configuration errors (e.g., `ModelNotAvailable`, `QuotaExceeded`).

5. **Backward Compatibility**: Adding a new enum variant is backward compatible. Existing code that matches on `LlmError::Api` will continue to work, though it won't benefit from the new variant until updated.

6. **Migration Path**:
   - Phase 1: Add `ProviderNotConfigured` variant to `LlmError`
   - Phase 2: Update `ProxyLlmClient` to detect and return the new variant
   - Phase 3: Add i18n strings for all supported locales
   - Phase 4: Update CLI error display to handle the new variant
   - Phase 5: Update Agent state machine to skip retries for this error type

## Follow-up Questions

- [ ] Should the CLI query `/health` on startup to validate provider availability before attempting LLM requests?
- [ ] Would a dedicated `/api/llm/capabilities` endpoint be valuable for exposing model lists, default mappings, and provider-specific features?

## Related Discoveries

- [[013-proxy-llm-client-503-handling]] — Documents the current 503 error handling that motivated this question
- [[021-cli-json-error-parsing]] — Proposes JSON parsing for cleaner error display, which complements this change
- [[014-provider-model-discovery]] — Discusses provider/model discovery mechanisms
