# Discovery: Should the `Retry-After` header use seconds (integer) or HTTP-date format for maximum compatibility?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The `Retry-After` header should use the seconds (integer) format, not HTTP-date.** This is the clear recommendation based on RFC specifications, existing Loom codebase patterns, industry practice from major API providers, and practical implementation considerations.

The seconds format (`delay-seconds`) is preferred because:

1. **RFC 9110 Preference**: RFC 9110 (Section 10.2.3, which obsoletes RFC 7231) defines two formats for `Retry-After`: `delay-seconds` (integer) and `HTTP-date`. For dynamic rate limiting scenarios, the seconds format is preferred because it avoids clock synchronization issues between client and server.

2. **Existing Loom Client Compatibility**: All existing Loom clients that parse `Retry-After` expect an integer format:
   - `loom-flags/src/client.rs` uses `.parse::<u64>().ok()`
   - `loom-analytics/src/client.rs` uses `.parse::<u64>().ok()`
   - `loom-server-llm-openai/src/client.rs` uses `.parse::<u64>().ok()`
   - `web/packages/http/src/client.ts` uses `parseInt(retryAfter, 10)`

3. **Major API Provider Practice**: OpenAI, Anthropic, and other major LLM providers use the seconds format for their `Retry-After` headers. The Loom codebase already parses these upstream headers as integers.

4. **Clock Skew Avoidance**: HTTP-date format requires synchronized clocks between client and server. In distributed systems with clients across different time zones and with potentially drifted clocks, the seconds format is more reliable.

5. **Simpler Implementation**: Integer parsing is simpler and less error-prone than HTTP-date parsing, which requires handling multiple date formats (RFC 1123, RFC 850, asctime).

## Evidence

**All Loom clients parse as integer** (`crates/loom-flags/src/client.rs:543-549`):
```rust
if response.status() == reqwest::StatusCode::TOO_MANY_REQUESTS {
    let retry_after = response
        .headers()
        .get("Retry-After")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok());  // Parses as integer
    return Err(FlagsError::RateLimited {
        retry_after_secs: retry_after,
    });
}
```

**OpenAI client parses as integer** (`crates/loom-server-llm-openai/src/client.rs:117-124`):
```rust
if status_code == 429 {
    let retry_after = response
        .headers()
        .get("retry-after")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse().ok());  // Parses as integer
    return LlmError::RateLimited { retry_after_secs: retry_after };
}
```

**Web client parses as integer** (`web/packages/http/src/client.ts:173-176`):
```typescript
if (response.status === 429) {
    const retryAfter = response.headers.get('Retry-After');
    throw new RateLimitError('Rate limited', {
        retryAfterSecs: retryAfter ? parseInt(retryAfter, 10) : undefined,  // Parses as integer
        body
    });
}
```

**Discovery 018 already recommends seconds** (`discoveries/018-retry-after-header-addition.md:118`):
> "Per RFC 9110, the `delay-seconds` format (integer) is preferred over `http-date` for dynamic rate limiting because it avoids clock skew issues between client and server."

**Proposed implementation uses integer** (`discoveries/030-servererror-ratelimited-implementation-complexity.md:73-76`):
```rust
if let Some(secs) = retry_after_secs {
    if let Ok(value) = HeaderValue::from_str(&secs.to_string()) {
        headers.insert("Retry-After", value);
    }
}
```

**Key Files:**
- `crates/loom-flags/src/client.rs` — Parses Retry-After as integer
- `crates/loom-analytics/src/client.rs` — Parses Retry-After as integer
- `crates/loom-server-llm-openai/src/client.rs` — Parses upstream Retry-After as integer
- `web/packages/http/src/client.ts` — Parses Retry-After as integer
- `crates/loom-server/src/error.rs` — Where ServerError::RateLimited will be implemented

## Implications

1. **Use `u64` for seconds**: The `retry_after_secs: Option<u64>` field type is correct and should be used consistently across the codebase.

2. **Header value format**: When setting the header, simply convert the integer to a string:
   ```rust
   headers.insert("Retry-After", HeaderValue::from_str(&secs.to_string()).unwrap());
   ```

3. **No HTTP-date parsing needed**: Clients do not need to implement HTTP-date parsing. If an upstream provider returns HTTP-date format (unlikely for LLM APIs), it will be ignored (parsed as `None`), and the client will fall back to default retry behavior.

4. **Backward compatibility**: Existing clients already expect integer format, so no changes are needed to client parsing logic.

5. **Maximum value consideration**: A `u64` can represent up to ~584 billion years in seconds, far exceeding any practical retry delay. However, for sanity, consider capping at a reasonable maximum (e.g., 86400 seconds = 24 hours) to prevent clients from waiting indefinitely due to server bugs.

6. **Consistency with error types**: The existing `LlmError::RateLimited { retry_after_secs: Option<u64> }` and similar error types already use `u64`, maintaining consistency.

## Follow-up Questions

- [ ] Should there be a maximum cap on `retry_after_secs` values (e.g., 3600 or 86400 seconds) to prevent unreasonably long waits?
- [ ] Should the server log a warning if upstream providers return HTTP-date format instead of seconds?

## Related Discoveries

- [[018-retry-after-header-addition]] — Documents the need for Retry-After header in responses
- [[030-servererror-ratelimited-implementation-complexity]] — Documents the implementation approach for ServerError::RateLimited
- [[009-proxy-llm-client-retry-after]] — Documents the need for Retry-After header parsing in ProxyLlmClient
