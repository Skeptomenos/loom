# Discovery: How does the CLI's ProxyLlmClient handle the 503 "provider not configured" error, and does it provide helpful user feedback?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The ProxyLlmClient handles 503 errors as generic API errors, providing moderately helpful but improvable user feedback.** When the server returns a 503 "provider not configured" error, the client wraps the entire HTTP response (status + body) into an `LlmError::Api` variant. This error is then displayed to the user via the i18n-formatted message `"Error: {error}"`, which includes the raw JSON error body from the server.

The current implementation has a key limitation: **the client only explicitly handles 429 (Too Many Requests) as a typed error (`LlmError::RateLimited`), while all other non-success status codes (including 503) are treated uniformly as `LlmError::Api`.** This means:

1. The user sees the full technical error: `"Error: API error: proxy returned status 503 Service Unavailable: {"error":"service_unavailable","message":"Anthropic provider is not configured on the server"}"`
2. The error message is informative (it includes the server's explanation) but not user-friendly (raw JSON in the output)
3. There's no special retry logic or actionable guidance for configuration errors

## Evidence

**ProxyLlmClient error handling** (`crates/loom-server-llm-proxy/src/client.rs:138-151`):

```rust
if !status.is_success() {
    let error_body = response.text().await.unwrap_or_default();
    debug!(status = %status, body = %error_body, "proxy returned error status");

    if status == reqwest::StatusCode::TOO_MANY_REQUESTS {
        return Err(LlmError::RateLimited {
            retry_after_secs: None,
        });
    }

    return Err(LlmError::Api(format!(
        "proxy returned status {status}: {error_body}"
    )));
}
```

**Server-side 503 generation** (`crates/loom-server/src/llm_proxy.rs:141-146`):

```rust
if !service.has_anthropic() {
    tracing::error!("proxy_anthropic_stream: Anthropic provider not configured");
    return Err(ServerError::ServiceUnavailable(
        "Anthropic provider is not configured on the server".into(),
    ));
}
```

**ServerError to HTTP mapping** (`crates/loom-server/src/error.rs:184-195`):

```rust
ServerError::ServiceUnavailable(msg) => {
    tracing::warn!(error = %msg, "service unavailable");
    (
        StatusCode::SERVICE_UNAVAILABLE,  // 503
        ErrorResponse {
            error: "service_unavailable".to_string(),
            message: msg.clone(),
            server_version: None,
            client_version: None,
        },
    )
}
```

**CLI error display** (`crates/loom-cli/src/main.rs:686-692`):

```rust
Err(e) => {
    error!(error = %e, "failed to start LLM request");
    eprintln!(
        "{}",
        loom_common_i18n::t_fmt(get_locale(), "client.repl.error", &[("error", &e.to_string())])
    );
}
```

**i18n error template** (`crates/loom-common-i18n/locales/en/messages.po:876-877`):

```
msgid "client.repl.error"
msgstr "Error: {error}"
```

**Key Files:**
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient implementation with HTTP error handling
- `crates/loom-server/src/llm_proxy.rs` — Server-side proxy routes that generate 503 errors
- `crates/loom-server/src/error.rs` — ServerError enum and HTTP status code mapping
- `crates/loom-common-core/src/error.rs` — LlmError enum definition
- `crates/loom-cli/src/main.rs` — CLI error display logic

## Implications

1. **User Experience**: Users see raw JSON in error messages, which is technically accurate but not polished. A user encountering "provider not configured" would benefit from actionable guidance like "Please configure the Anthropic API key on the server."

2. **Error Classification**: The lack of a dedicated error variant for "provider not configured" means the CLI cannot distinguish between:
   - Configuration errors (fixable by admin)
   - Rate limits (retry later)
   - Transient server issues (retry immediately)
   - Permanent API errors (investigate)

3. **Retry Logic**: Since 503 is not recognized as a retryable error, the Agent state machine won't automatically retry when a provider is temporarily unavailable (e.g., during server restart).

4. **Semantic Mismatch**: As noted in discovery-009, the server maps rate limits to 503, but the client only recognizes 429 for rate-limit handling. This creates inconsistent behavior.

## Follow-up Questions

- [ ] Should the CLI parse the JSON error response to extract and display just the `message` field for cleaner user output?
- [ ] Would adding a `LlmError::ProviderNotConfigured` variant improve error handling and enable provider-specific guidance in the CLI?

## Related Discoveries

- [[009-proxy-llm-client-retry-after]] — Documents the semantic mismatch between server 503 and client 429 handling for rate limits
- [[006-llm-service-model-resolution]] — Original source of this follow-up question
- [[003-request-flow]] — Documents the complete request flow through the proxy
