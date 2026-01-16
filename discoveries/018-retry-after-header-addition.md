# Discovery: Should the Retry-After header be added to the HTTP response in addition to the message body?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the `Retry-After` header should be added to HTTP responses for rate-limited requests.** Currently, the server embeds retry timing information only in the JSON message body (e.g., "LLM rate limited; retry after 30 seconds"), but does not set the standard HTTP `Retry-After` header. This is a missed opportunity because:

1. **HTTP Standard Compliance**: RFC 9110 (Section 10.2.3, obsoleting RFC 7231) defines `Retry-After` as the standard mechanism for servers to indicate when clients should retry. RFC 6585 explicitly recommends including it with 429 responses.

2. **Machine-Parseable**: The header provides a machine-readable signal that clients can use for automatic retry logic, without needing to parse natural language from the message body.

3. **Existing Client Support**: The Loom codebase already has clients that parse `Retry-After`:
   - `loom-flags/src/client.rs` parses `Retry-After` from 429 responses
   - `loom-analytics/src/client.rs` parses `Retry-After` from 429 responses
   - `web/packages/http/src/client.ts` parses `Retry-After` from 429 responses
   - `web/packages/flags/src/client.ts` parses `Retry-After` from 429 responses

4. **Upstream Providers Use It**: The OpenAI client (`loom-server-llm-openai`) already parses `retry-after` from upstream responses. The server should propagate this information downstream.

The implementation requires modifying the `IntoResponse` implementation for `ServerError` to return custom headers alongside the JSON body. Axum supports this via tuple responses `(StatusCode, HeaderMap, Json<T>)`.

## Evidence

**Server currently embeds retry info in message only** (`crates/loom-server/src/llm_proxy.rs:418-424`):
```rust
LlmError::RateLimited { retry_after_secs } => {
    tracing::warn!(retry_after = ?retry_after_secs, "LLM rate limited");
    let msg = match retry_after_secs {
        Some(secs) => format!("LLM rate limited; retry after {secs} seconds"),
        None => "LLM rate limited; try again later".to_string(),
    };
    ServerError::ServiceUnavailable(msg)  // No Retry-After header set
}
```

**ServerError::ServiceUnavailable returns only JSON body** (`crates/loom-server/src/error.rs:184-195`):
```rust
ServerError::ServiceUnavailable(msg) => {
    tracing::warn!(error = %msg, "service unavailable");
    (
        StatusCode::SERVICE_UNAVAILABLE,
        ErrorResponse {
            error: "service_unavailable".to_string(),
            message: msg.clone(),
            server_version: None,
            client_version: None,
        },
    )
}
// Note: Returns (StatusCode, Json<ErrorResponse>) - no headers
```

**Clients already parse Retry-After** (`crates/loom-flags/src/client.rs:542-551`):
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

**Web client also parses it** (`web/packages/http/src/client.ts:172-178`):
```typescript
if (response.status === 429) {
    const retryAfter = response.headers.get('Retry-After');
    throw new RateLimitError('Rate limited', {
        retryAfterSecs: retryAfter ? parseInt(retryAfter, 10) : undefined,
        body
    });
}
```

**OpenAI client parses upstream header** (`crates/loom-server-llm-openai/src/client.rs:117-124`):
```rust
if status_code == 429 {
    let retry_after = response
        .headers()
        .get("retry-after")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse().ok());
    return LlmError::RateLimited { retry_after_secs: retry_after };
}
```

**Key Files:**
- `crates/loom-server/src/error.rs` — ServerError enum and IntoResponse impl (needs modification)
- `crates/loom-server/src/llm_proxy.rs` — LLM error mapping (needs to pass retry_after_secs)
- `crates/loom-flags/src/client.rs` — Example of correct Retry-After parsing
- `crates/loom-analytics/src/client.rs` — Example of correct Retry-After parsing
- `web/packages/http/src/client.ts` — Web client Retry-After parsing

## Implications

1. **New ServerError variant needed**: As identified in discovery 017, a `ServerError::RateLimited { message: String, retry_after_secs: Option<u64> }` variant should be added that:
   - Returns HTTP 429 (not 503)
   - Sets the `Retry-After` header when `retry_after_secs` is `Some`
   - Includes the retry info in the JSON body for backward compatibility

2. **Implementation approach**: Modify `IntoResponse` for `ServerError` to return `(StatusCode, HeaderMap, Json<ErrorResponse>)` for the `RateLimited` variant:
   ```rust
   ServerError::RateLimited { message, retry_after_secs } => {
       let mut headers = HeaderMap::new();
       if let Some(secs) = retry_after_secs {
           headers.insert("Retry-After", HeaderValue::from(secs));
       }
       (StatusCode::TOO_MANY_REQUESTS, headers, Json(ErrorResponse { ... }))
   }
   ```

3. **Use seconds format**: Per RFC 9110, the `delay-seconds` format (integer) is preferred over `http-date` for dynamic rate limiting because it avoids clock skew issues between client and server.

4. **Consider jitter**: When many clients are rate-limited simultaneously, adding small random jitter to the `Retry-After` value can prevent "thundering herd" retries.

5. **Backward compatibility**: The JSON body should continue to include the retry information for clients that don't parse headers.

## Follow-up Questions

- [ ] Should the ErrorResponse struct be extended with an optional `retry_after_secs` field for explicit machine-readable retry timing in the body?
- [ ] Should jitter be added to Retry-After values to prevent thundering herd retries?

## Related Discoveries

- [[017-http-429-vs-503-rate-limiting]] — Documents the need for 429 instead of 503 for rate limits
- [[009-proxy-llm-client-retry-after]] — Documents the need for Retry-After header parsing in ProxyLlmClient
- [[013-proxy-llm-client-503-handling]] — Documents the semantic mismatch between server 503 and client 429 handling
