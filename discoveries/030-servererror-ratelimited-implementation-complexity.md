# Discovery: What is the implementation complexity of adding a `ServerError::RateLimited` variant with `Retry-After` header support?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The implementation complexity is LOW to MEDIUM.** Adding a `ServerError::RateLimited` variant with `Retry-After` header support requires changes to 2-3 files with approximately 30-50 lines of code. The main complexity comes from modifying the `IntoResponse` implementation to return custom headers, which requires a different return type for one variant.

The implementation involves:
1. **Adding a new enum variant** to `ServerError` (~5 lines)
2. **Modifying `IntoResponse`** to handle the new variant with custom headers (~20 lines)
3. **Updating call sites** in `llm_proxy.rs` and other routes (~10 lines across 4 files)

The codebase already has patterns for returning custom headers in responses (e.g., `loom-server-auth/src/middleware.rs`), so this is a well-understood pattern. The main consideration is that the `IntoResponse` implementation must return a `Response` type directly (not a tuple) when different variants need different header sets.

## Evidence

**Current ServerError enum** (`crates/loom-server/src/error.rs:16-72`):
```rust
#[derive(Debug, thiserror::Error)]
pub enum ServerError {
    // ... existing variants ...
    #[error("Service unavailable: {0}")]
    ServiceUnavailable(String),
    // ... more variants ...
}
```

**Current IntoResponse returns tuple** (`crates/loom-server/src/error.rs:85-276`):
```rust
impl IntoResponse for ServerError {
    fn into_response(self) -> Response {
        let (status, error_response) = match &self {
            // All variants return (StatusCode, ErrorResponse)
            ServerError::ServiceUnavailable(msg) => {
                (StatusCode::SERVICE_UNAVAILABLE, ErrorResponse { ... })
            }
            // ...
        };
        (status, Json(error_response)).into_response()
    }
}
```

**Existing pattern for custom headers** (`crates/loom-server/src/routes/auth.rs:210`):
```rust
let mut headers = HeaderMap::new();
headers.insert(
    header::SET_COOKIE,
    HeaderValue::from_str(&cookie_value).unwrap(),
);
(StatusCode::OK, headers, Json(response)).into_response()
```

**Axum supports 3-tuple responses**: `(StatusCode, HeaderMap, Json<T>)` implements `IntoResponse`.

**Proposed new variant**:
```rust
/// Rate limited (too many requests).
#[error("Rate limited: {message}")]
RateLimited {
    message: String,
    retry_after_secs: Option<u64>,
},
```

**Proposed IntoResponse modification**:
```rust
ServerError::RateLimited { message, retry_after_secs } => {
    let mut headers = HeaderMap::new();
    if let Some(secs) = retry_after_secs {
        if let Ok(value) = HeaderValue::from_str(&secs.to_string()) {
            headers.insert("Retry-After", value);
        }
    }
    let error_response = ErrorResponse {
        error: "rate_limited".to_string(),
        message: message.clone(),
        server_version: None,
        client_version: None,
    };
    return (StatusCode::TOO_MANY_REQUESTS, headers, Json(error_response)).into_response();
}
```

**Call sites that need updating**:
- `crates/loom-server/src/llm_proxy.rs:418-424` — LLM rate limits
- `crates/loom-server/src/routes/cse.rs:84-86` — Google CSE rate limits
- `crates/loom-server/src/routes/github.rs:476-478` — GitHub rate limits
- `crates/loom-server/src/routes/serper.rs:70-72` — Serper rate limits

**Key Files:**
- `crates/loom-server/src/error.rs` — Add variant and modify IntoResponse
- `crates/loom-server/src/llm_proxy.rs` — Update map_llm_error
- `crates/loom-server/src/routes/cse.rs` — Update error mapping
- `crates/loom-server/src/routes/github.rs` — Update error mapping
- `crates/loom-server/src/routes/serper.rs` — Update error mapping

## Implications

1. **Low risk change**: The modification is additive—existing variants are unchanged. Only the new `RateLimited` variant uses custom headers.

2. **IntoResponse refactoring**: The current implementation builds a `(status, error_response)` tuple and converts at the end. The `RateLimited` variant needs an early return to include headers. This is a minor structural change.

3. **Header value safety**: `HeaderValue::from_str` can fail for non-ASCII characters, but since we're converting a `u64` to string, this is safe. The code should still handle the error gracefully.

4. **Backward compatibility**: Clients that don't parse the `Retry-After` header will still receive the retry information in the JSON body (via the `message` field). Consider also adding an optional `retry_after_secs` field to `ErrorResponse` for machine-readable access.

5. **Testing**: Add unit tests for the new variant, including:
   - 429 status code is returned
   - `Retry-After` header is set when `retry_after_secs` is `Some`
   - `Retry-After` header is absent when `retry_after_secs` is `None`
   - JSON body contains correct error code and message

6. **Consistency across routes**: All rate-limit scenarios should use the new variant for consistent behavior. This includes LLM proxy, Google CSE, GitHub, and Serper routes.

## Follow-up Questions

- [ ] Should the `ErrorResponse` struct include an optional `retry_after_secs: Option<u64>` field for explicit machine-readable retry timing in the JSON body?
- [ ] Should the implementation include jitter (e.g., `retry_after_secs + rand(0..5)`) to prevent thundering herd retries?

## Related Discoveries

- [[017-http-429-vs-503-rate-limiting]] — Documents the need for 429 instead of 503 for rate limits
- [[018-retry-after-header-addition]] — Documents the need for Retry-After header in responses
- [[009-proxy-llm-client-retry-after]] — Documents the need for Retry-After header parsing in ProxyLlmClient
