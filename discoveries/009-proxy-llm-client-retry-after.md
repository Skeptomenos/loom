# Discovery: Could the ProxyLlmClient be enhanced to parse the Retry-After header for rate-limited responses to enable smarter retry timing?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the ProxyLlmClient could and should be enhanced to parse the `Retry-After` header.** Currently, when the proxy returns a 429 response, the client creates an `LlmError::RateLimited { retry_after_secs: None }`, discarding any timing information the server might have provided. This is a missed opportunity because:

1. **The server-side already has the information**: The OpenAI client (`loom-server-llm-openai`) parses the `retry-after` header and includes it in `LlmError::RateLimited`. The Anthropic pool also tracks cooldown periods (2 hours for quota exhaustion).

2. **The server-side proxy includes it in error messages**: When `map_llm_error` converts `LlmError::RateLimited` to `ServerError::ServiceUnavailable`, it includes the retry timing in the message text (e.g., "LLM rate limited; retry after 30 seconds"), but this is not machine-parseable by the client.

3. **Other Loom clients already do this**: The `loom-flags` client and `loom-analytics` client both parse the `Retry-After` header from 429 responses and include it in their error types.

The enhancement would be straightforward: extract the `Retry-After` header from the `reqwest::Response` before consuming the body, similar to how `loom-flags` and `loom-analytics` do it.

## Evidence

**ProxyLlmClient currently ignores Retry-After** (`crates/loom-server-llm-proxy/src/client.rs:138-146`):
```rust
if !status.is_success() {
    let error_body = response.text().await.unwrap_or_default();
    
    if status == reqwest::StatusCode::TOO_MANY_REQUESTS {
        return Err(LlmError::RateLimited {
            retry_after_secs: None,  // <-- Always None, header not parsed
        });
    }
    // ...
}
```

**OpenAI server-side client parses it correctly** (`crates/loom-server-llm-openai/src/client.rs:116-126`):
```rust
if status_code == 429 {
    let retry_after = response
        .headers()
        .get("retry-after")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse().ok());

    return LlmError::RateLimited {
        retry_after_secs: retry_after,  // <-- Properly extracted
    };
}
```

**loom-flags client parses it** (`crates/loom-flags/src/client.rs:542-550`):
```rust
if response.status() == reqwest::StatusCode::TOO_MANY_REQUESTS {
    let retry_after = response
        .headers()
        .get("Retry-After")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok());
    return Err(FlagsError::RateLimited {
        retry_after_secs: retry_after,
    });
}
```

**loom-analytics client parses it** (`crates/loom-analytics/src/client.rs:236-244`):
```rust
if response.status() == reqwest::StatusCode::TOO_MANY_REQUESTS {
    let retry_after = response
        .headers()
        .get("Retry-After")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok());
    return Err(AnalyticsError::RateLimited {
        retry_after_secs: retry_after,
    });
}
```

**Server includes retry info in message but not as header** (`crates/loom-server/src/llm_proxy.rs:418-424`):
```rust
LlmError::RateLimited { retry_after_secs } => {
    tracing::warn!(retry_after = ?retry_after_secs, "LLM rate limited");
    let msg = match retry_after_secs {
        Some(secs) => format!("LLM rate limited; retry after {secs} seconds"),
        None => "LLM rate limited; try again later".to_string(),
    };
    ServerError::ServiceUnavailable(msg)  // <-- Message only, no header
}
```

**Key Files:**
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient implementation (needs enhancement)
- `crates/loom-server-llm-openai/src/client.rs` — Example of correct Retry-After parsing
- `crates/loom-flags/src/client.rs` — Another example of correct parsing
- `crates/loom-analytics/src/client.rs` — Another example of correct parsing
- `crates/loom-server/src/llm_proxy.rs` — Server-side error mapping (could add header)
- `crates/loom-server/src/error.rs` — ServerError::ServiceUnavailable (returns 503, not 429)

## Implications

- **Two-part fix needed**: For full Retry-After support, both the server and client need changes:
  1. **Server**: Modify `ServerError::ServiceUnavailable` to optionally include a `Retry-After` header in the HTTP response (currently it only includes the info in the message body)
  2. **Client**: Modify `ProxyLlmClient` to parse the `Retry-After` header before consuming the response body

- **Server returns 503, not 429**: The `map_llm_error` function converts `LlmError::RateLimited` to `ServerError::ServiceUnavailable`, which returns HTTP 503. The client currently only checks for 429. This needs alignment—either the server should return 429 for rate limits, or the client should also handle 503 with Retry-After.

- **Consistent pattern exists**: The codebase already has a consistent pattern for parsing Retry-After in `loom-flags` and `loom-analytics`. The ProxyLlmClient should follow the same pattern.

- **Enables smarter client-side backoff**: With the retry timing available, the CLI or Agent runtime could use the provider's suggested wait time instead of a generic exponential backoff, reducing unnecessary retries and improving user experience.

## Follow-up Questions

- [ ] Should the server return 429 instead of 503 for rate-limited responses to maintain semantic correctness?
- [ ] Should the `Retry-After` header be added to the HTTP response in addition to the message body?

## Related Discoveries

- [[008-retry-timeout-mechanism]] — Agent-level retry architecture (currently unused)
- [[007-sse-error-handling]] — How SSE errors propagate to clients
- [[006-llm-service-model-resolution]] — LlmService architecture
