# Discovery: Should the CLI parse the JSON error response to extract and display just the message field for cleaner user output?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the CLI should parse the JSON error response to extract the `message` field for significantly cleaner user output.** The current implementation displays raw JSON to users, which is technically accurate but provides a poor user experience. The implementation would be straightforward with minimal complexity.

Currently, when a user encounters a server error (e.g., "provider not configured"), they see:
```
Error: API error: proxy returned status 503 Service Unavailable: {"error":"service_unavailable","message":"Anthropic provider is not configured on the server"}
```

With JSON parsing, they would see:
```
Error: Anthropic provider is not configured on the server
```

The implementation is low-risk because:
1. The server already returns a well-defined `ErrorResponse` struct with consistent `error` and `message` fields
2. Fallback to raw body display is trivial if JSON parsing fails
3. The change is isolated to the `ProxyLlmClient` in a single file
4. No breaking changes to the `LlmError` enum are required

## Evidence

**Server ErrorResponse structure** (`crates/loom-server/src/error.rs:76-83`):
```rust
pub struct ErrorResponse {
    pub error: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub server_version: Option<u64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub client_version: Option<u64>,
}
```

**Current ProxyLlmClient error handling** (`crates/loom-server-llm-proxy/src/client.rs:138-151`):
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

**Proposed implementation** (minimal change):
```rust
// Add to types.rs or client.rs
#[derive(Deserialize)]
struct ProxyErrorResponse {
    #[allow(dead_code)]
    error: String,
    message: String,
}

// In error handling block:
if !status.is_success() {
    let error_body = response.text().await.unwrap_or_default();
    debug!(status = %status, body = %error_body, "proxy returned error status");

    if status == reqwest::StatusCode::TOO_MANY_REQUESTS {
        return Err(LlmError::RateLimited {
            retry_after_secs: None,
        });
    }

    // Try to extract clean message from JSON, fallback to raw body
    let message = serde_json::from_str::<ProxyErrorResponse>(&error_body)
        .map(|r| r.message)
        .unwrap_or_else(|_| format!("proxy returned status {status}: {error_body}"));

    return Err(LlmError::Api(message));
}
```

**Existing pattern in codebase** - The `WeaverClient` already uses a similar pattern for error handling (`crates/loom-cli/src/weaver_client.rs:95-99`):
```rust
if !response.status().is_success() {
    let status = response.status();
    let body = response.text().await.unwrap_or_default();
    anyhow::bail!("Failed to create weaver: {status} - {body}");
}
```

This could also benefit from JSON parsing, showing the pattern is consistently used but not yet optimized.

**Key Files:**
- `crates/loom-server-llm-proxy/src/client.rs` - ProxyLlmClient where parsing would be added
- `crates/loom-server-llm-proxy/src/types.rs` - Where ProxyErrorResponse struct could be defined
- `crates/loom-server/src/error.rs` - Server-side ErrorResponse definition (reference)
- `crates/loom-cli/src/weaver_client.rs` - Similar pattern that could also be improved

## Implications

1. **User Experience**: Users will see clean, actionable error messages instead of raw JSON. This is especially important for configuration errors where the message contains guidance.

2. **Consistency**: The `error` code field could be preserved in debug logs while showing only the `message` to users, maintaining debuggability while improving UX.

3. **Extensibility**: Once the `ProxyErrorResponse` struct exists, additional fields (like `retry_after_secs` from discovery-018) can be easily extracted.

4. **Minimal Risk**: The fallback to raw body display ensures no regression if the server returns non-JSON errors or the format changes.

5. **Pattern Propagation**: The same improvement could be applied to `WeaverClient` and other HTTP clients in the CLI for consistent error display.

## Follow-up Questions

- [ ] Should the `error` code field be included in the displayed message for certain error types (e.g., "conflict: Version mismatch...")?
- [ ] Should the CLI implement a shared `parse_error_response()` helper function for use across WeaverClient, ProxyLlmClient, and future HTTP clients?

## Related Discoveries

- [[013-proxy-llm-client-503-handling]] - Documents the current 503 error handling that motivated this question
- [[018-retry-after-header-addition]] - Proposes adding retry_after_secs to error responses, which would benefit from JSON parsing
