# Discovery: Should the ErrorResponse struct be extended with an optional `retry_after_secs` field?

> Category: Follow-ups (Error Handling / Rate Limiting)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the `ErrorResponse` struct should be extended with an optional `retry_after_secs: Option<u64>` field.** This addition provides machine-readable retry timing in the JSON response body, complementing the `Retry-After` HTTP header proposed in discovery 018. The dual approach (header + body field) ensures maximum compatibility:

1. **Header-aware clients** can use the standard `Retry-After` HTTP header for immediate retry scheduling
2. **Simple clients** that only parse JSON bodies can extract `retry_after_secs` without header parsing
3. **Logging and debugging** benefit from having retry timing visible in the serialized error response

The current `ErrorResponse` struct embeds retry information only in the human-readable `message` field (e.g., "LLM rate limited; retry after 30 seconds"), which requires fragile string parsing. A dedicated field eliminates this anti-pattern.

## Evidence

### Current ErrorResponse Structure

**File**: `crates/loom-server/src/error.rs`

```rust
#[derive(Debug, Serialize, ToSchema)]
pub struct ErrorResponse {
    pub error: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub server_version: Option<u64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub client_version: Option<u64>,
}
```

The struct already uses optional fields with `skip_serializing_if` for context-specific data (`server_version`, `client_version` for conflict errors). Adding `retry_after_secs` follows this established pattern.

### Current Rate Limit Handling (Embeds in Message)

**File**: `crates/loom-server/src/llm_proxy.rs` (lines 418-425)

```rust
LlmError::RateLimited { retry_after_secs } => {
    tracing::warn!(retry_after = ?retry_after_secs, "LLM rate limited");
    let msg = match retry_after_secs {
        Some(secs) => format!("LLM rate limited; retry after {secs} seconds"),
        None => "LLM rate limited; try again later".to_string(),
    };
    ServerError::ServiceUnavailable(msg)
}
```

The `retry_after_secs` value is available but only embedded in the message string, making it inaccessible for programmatic use.

### Existing Pattern in Client Error Types

Multiple client SDKs already define `retry_after_secs` fields in their error types:

**File**: `crates/loom-common-core/src/error.rs`
```rust
#[error("Rate limited: retry after {retry_after_secs:?} seconds")]
RateLimited { retry_after_secs: Option<u64> },
```

**File**: `crates/loom-analytics/src/error.rs`
```rust
#[error("rate limited, retry after {retry_after_secs:?} seconds")]
RateLimited { retry_after_secs: Option<u64> },
```

**File**: `crates/loom-flags/src/error.rs`
```rust
#[error("Rate limited. Retry after {retry_after_secs:?} seconds")]
RateLimited { retry_after_secs: Option<u64> },
```

This demonstrates the codebase already uses `u64` for retry timing consistently.

### Proposed Implementation

```rust
#[derive(Debug, Serialize, ToSchema)]
pub struct ErrorResponse {
    pub error: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub server_version: Option<u64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub client_version: Option<u64>,
    /// Seconds until the client should retry (for rate-limited responses).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub retry_after_secs: Option<u64>,
}
```

The `IntoResponse` implementation for `ServerError::RateLimited` (once added per discovery 030) would populate this field:

```rust
ServerError::RateLimited { message, retry_after_secs } => {
    // Set Retry-After header (per discovery 018)
    let mut headers = HeaderMap::new();
    if let Some(secs) = retry_after_secs {
        headers.insert("Retry-After", HeaderValue::from(secs));
    }
    
    (
        StatusCode::TOO_MANY_REQUESTS,
        headers,
        Json(ErrorResponse {
            error: "rate_limited".to_string(),
            message,
            server_version: None,
            client_version: None,
            retry_after_secs,  // Machine-readable in body
        }),
    )
}
```

**Key Files:**
- `crates/loom-server/src/error.rs` — Add field to `ErrorResponse`, update `IntoResponse`
- `crates/loom-server/src/llm_proxy.rs` — Update `map_llm_error` to use new `RateLimited` variant
- `crates/loom-server/src/api_response.rs` — Update `ApiErrorResponse` trait if needed
- `specs/api-documentation.md` — Update OpenAPI schema documentation

## Implications

1. **Backward Compatibility**: The field uses `skip_serializing_if = "Option::is_none"`, so existing clients that don't expect this field will not see it in non-rate-limited responses. This is a non-breaking change.

2. **Client Updates**: The `ProxyLlmClient` in `loom-server-llm-proxy` should be updated to parse this field from the JSON body as a fallback when the `Retry-After` header is missing (some proxies strip headers).

3. **Consistency**: All rate-limited error responses should populate both the header AND the body field when `retry_after_secs` is `Some`. This provides defense-in-depth for retry timing propagation.

4. **OpenAPI Documentation**: The `ErrorResponse` schema in the OpenAPI spec should document this new optional field with its semantics.

5. **Testing**: Add tests verifying that rate-limited responses include `retry_after_secs` in the JSON body when the value is known.

## Follow-up Questions

- [ ] Should the `ApiErrorResponse` trait and `impl_api_error_response!` macro be updated to support `retry_after_secs` for domain-specific error types (e.g., `OrgErrorResponse`, `RepoErrorResponse`)?
- [ ] Should clients implement a fallback strategy that checks the JSON body's `retry_after_secs` when the `Retry-After` header is missing or unparseable?

## Related Discoveries

- [[018-retry-after-header-addition]] — Proposes adding `Retry-After` HTTP header (this discovery complements it with body field)
- [[030-servererror-ratelimited-implementation-complexity]] — Proposes `ServerError::RateLimited` variant that would populate this field
- [[031-retry-after-header-format]] — Confirms `u64` seconds format for consistency
- [[021-cli-json-error-parsing]] — Notes that JSON parsing infrastructure would benefit from this field
