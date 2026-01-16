# Discovery: Should the server return 429 instead of 503 for rate-limited responses to maintain semantic correctness?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the server should return 429 (Too Many Requests) instead of 503 (Service Unavailable) for rate-limited responses.** The current implementation in `map_llm_error` converts `LlmError::RateLimited` to `ServerError::ServiceUnavailable`, which returns HTTP 503. This is semantically incorrect and causes practical problems:

1. **HTTP Semantics**: RFC 6585 defines 429 specifically for rate limiting ("the user has sent too many requests in a given amount of time"). HTTP 503 means the server is temporarily unable to handle the request due to overload or maintenance—a different condition entirely.

2. **Client Confusion**: The `ProxyLlmClient` only checks for 429 to trigger `LlmError::RateLimited`. When the server returns 503, the client treats it as a generic API error, losing the rate-limit semantics and the `retry_after_secs` information.

3. **Retry Logic Mismatch**: The shared retry logic in `loom-common-http` treats both 429 and 503 as retryable, but with different semantics. A 429 with `Retry-After` should use the server-provided delay, while a 503 typically uses exponential backoff.

4. **Industry Standard**: Major LLM providers (OpenAI, Anthropic) return 429 for rate limits, and the Loom server-side clients (OpenAI, Anthropic) correctly detect 429 and parse `Retry-After`. The proxy should maintain this semantic consistency.

The fix requires introducing a new `ServerError` variant (e.g., `RateLimited`) that returns HTTP 429 with an optional `Retry-After` header.

## Evidence

**Current mapping loses rate-limit semantics** (`crates/loom-server/src/llm_proxy.rs:418-424`):
```rust
LlmError::RateLimited { retry_after_secs } => {
    tracing::warn!(retry_after = ?retry_after_secs, "LLM rate limited");
    let msg = match retry_after_secs {
        Some(secs) => format!("LLM rate limited; retry after {secs} seconds"),
        None => "LLM rate limited; try again later".to_string(),
    };
    ServerError::ServiceUnavailable(msg)  // <-- Returns 503, not 429
}
```

**ServerError::ServiceUnavailable returns 503** (`crates/loom-server/src/error.rs:184-195`):
```rust
ServerError::ServiceUnavailable(msg) => {
    tracing::warn!(error = %msg, "service unavailable");
    (
        StatusCode::SERVICE_UNAVAILABLE,  // HTTP 503
        ErrorResponse {
            error: "service_unavailable".to_string(),
            message: msg.clone(),
            // No Retry-After header is set
            ...
        },
    )
}
```

**ProxyLlmClient only checks for 429** (`crates/loom-server-llm-proxy/src/client.rs:142-146`):
```rust
if status == reqwest::StatusCode::TOO_MANY_REQUESTS {  // Only 429
    return Err(LlmError::RateLimited {
        retry_after_secs: None,  // Also doesn't parse header
    });
}
// 503 falls through to generic LlmError::Api
```

**Weaver provisioner correctly uses 429** (`crates/loom-server/src/error.rs:241-249`):
```rust
ProvisionerError::TooManyWeavers { current, max } => (
    StatusCode::TOO_MANY_REQUESTS,  // Correct: 429
    ErrorResponse {
        error: "too_many_weavers".to_string(),
        message: format!("{current} weavers running (max: {max})"),
        ...
    },
),
```

**Other routes also misuse 503 for rate limits** (`crates/loom-server/src/routes/cse.rs:84-86`, `routes/github.rs:476-478`):
```rust
CseError::RateLimited => {
    ServerError::ServiceUnavailable("Google CSE rate limit exceeded; try again later".into())
}
// Same pattern in GitHub routes
```

**Key Files:**
- `crates/loom-server/src/llm_proxy.rs` — LLM error mapping (needs fix)
- `crates/loom-server/src/error.rs` — ServerError enum (needs new variant)
- `crates/loom-server/src/routes/cse.rs` — CSE rate limit handling (needs fix)
- `crates/loom-server/src/routes/github.rs` — GitHub rate limit handling (needs fix)
- `crates/loom-server/src/routes/serper.rs` — Serper rate limit handling (needs fix)
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient (needs to handle both 429 and parse Retry-After)

## Implications

1. **New ServerError variant needed**: Add `ServerError::RateLimited { message: String, retry_after_secs: Option<u64> }` that returns HTTP 429 with an optional `Retry-After` header.

2. **Multiple routes need updating**: The fix should be applied consistently across:
   - `llm_proxy.rs` — LLM rate limits
   - `routes/cse.rs` — Google CSE rate limits
   - `routes/github.rs` — GitHub API rate limits
   - `routes/serper.rs` — Serper rate limits

3. **Client needs dual handling**: The `ProxyLlmClient` should:
   - Continue checking for 429 (correct behavior)
   - Also check for 503 with rate-limit indicators (backward compatibility during transition)
   - Parse the `Retry-After` header from the response

4. **Retry-After header propagation**: The server should set the `Retry-After` header in the HTTP response, not just embed it in the message body. This enables machine-parseable retry timing.

5. **Backward compatibility**: Existing clients that don't understand 429 will see it as a client error (4xx) rather than a server error (5xx). This is actually more correct—rate limiting is a client-side concern (too many requests from this client), not a server-side failure.

## Follow-up Questions

- [ ] What is the implementation complexity of adding a `ServerError::RateLimited` variant with `Retry-After` header support?
- [ ] Should the `Retry-After` header use seconds (integer) or HTTP-date format for maximum compatibility?

## Related Discoveries

- [[009-proxy-llm-client-retry-after]] — Documents the need for Retry-After header parsing in ProxyLlmClient
- [[013-proxy-llm-client-503-handling]] — Documents the semantic mismatch between server 503 and client 429 handling
- [[008-retry-timeout-mechanism]] — Agent-level retry architecture
