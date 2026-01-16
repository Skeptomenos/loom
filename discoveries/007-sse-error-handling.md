# Discovery: How does the SSE error handling work when the connection drops mid-stream, and how does the client recover?

> Category: Follow-up (Data Flow)
> Discovered: 2026-01-16
> Confidence: High

## Answer

SSE error handling in Loom operates at **three distinct layers**, each with different responsibilities and recovery mechanisms. When a connection drops mid-stream, the error propagates upward through these layers, but **there is no automatic reconnection or stream resumption** at the proxy client level. Recovery relies on the Agent state machine's retry mechanism at the application layer.

**Layer 1 - ProxyLlmStream (Wire Level)**: The `ProxyLlmStream` in `loom-server-llm-proxy` handles raw byte stream parsing. When the underlying `reqwest` stream yields an error (e.g., connection reset, timeout), it emits an `LlmEvent::Error(LlmError::Http(...))`. If the stream ends with unparsed data in the buffer (partial SSE event), it logs a debug message but discards the incomplete data since SSE requires `\n\n` to terminate events.

**Layer 2 - ProxyLlmClient (HTTP Level)**: The client handles pre-stream errors (connection failures, non-2xx status codes) by returning `LlmError` immediately. For 429 (rate limited), it returns `LlmError::RateLimited`. The client does **not** implement automatic retries for streaming requests, even though `loom_common_http::retry` exists for non-streaming operations.

**Layer 3 - Agent State Machine (Application Level)**: The Agent in `loom-common-core` provides the actual recovery mechanism. When it receives `LlmEvent::Error`, it transitions to an `Error` state with a retry counter. If retries remain (< `max_retries`), it waits for a `RetryTimeoutFired` event and then re-sends the entire LLM request from the beginning. This is a **full restart**, not a resume from the last received delta.

## Evidence

**ProxyLlmStream error handling** (`crates/loom-server-llm-proxy/src/stream.rs:145-148`):
```rust
Poll::Ready(Some(Err(e))) => {
    warn!(error = %e, "SSE stream error");
    return Poll::Ready(Some(LlmEvent::Error(LlmError::Http(e.to_string()))));
}
```

**ProxyLlmStream partial data handling** (`crates/loom-server-llm-proxy/src/stream.rs:149-154`):
```rust
Poll::Ready(None) => {
    if !this.buffer.is_empty() {
        debug!(remaining = %this.buffer, "SSE stream ended with unparsed data");
    }
    return Poll::Ready(None);
}
```

**ProxyLlmClient HTTP error handling** (`crates/loom-server-llm-proxy/src/client.rs:173-194`):
```rust
let response = req.send().await.map_err(|e| {
    debug!(error = %e, "HTTP request failed");
    LlmError::Http(e.to_string())
})?;

if !status.is_success() {
    if status == reqwest::StatusCode::TOO_MANY_REQUESTS {
        return Err(LlmError::RateLimited { retry_after_secs: None });
    }
    return Err(LlmError::Api(format!("proxy returned status {status}: {error_body}")));
}
```

**Agent retry mechanism** (`crates/loom-common-core/src/agent.rs:214-252`):
```rust
// CallingLlm + LlmEvent::Error -> Error state with retry
(AgentState::CallingLlm { conversation, retries }, AgentEvent::LlmEvent(LlmEvent::Error(e))) => {
    let new_retries = *retries + 1;
    if new_retries < self.config.max_retries {
        self.state = AgentState::Error { conversation, error, retries: new_retries, origin: ErrorOrigin::Llm };
        AgentAction::WaitForInput  // Wait for RetryTimeoutFired
    } else {
        AgentAction::DisplayError(AgentError::Llm(e).to_string())
    }
}
```

**CLI error logging** (`crates/loom-cli/src/main.rs:579-581`):
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
}
```

**Key Files:**
- `crates/loom-server-llm-proxy/src/stream.rs` - SSE parsing and stream-level error handling
- `crates/loom-server-llm-proxy/src/client.rs` - HTTP client and pre-stream error handling
- `crates/loom-common-core/src/agent.rs` - Agent state machine with retry logic
- `crates/loom-common-core/src/error.rs` - LlmError type definitions
- `crates/loom-common-http/src/retry.rs` - Retry utility (not used for streaming)

## Implications

- **No automatic reconnection**: If a stream drops, the entire LLM request must be re-sent. Any tokens already received are lost from the streaming perspective (though they may have been displayed to the user).

- **Retry is at message level, not token level**: The Agent retries by sending the full conversation history again. This is safe because LLM responses are deterministic given the same input (mostly), but it wastes tokens and time.

- **Rate limiting is detected but not gracefully handled**: The client detects 429 responses and returns `LlmError::RateLimited`, but the `retry_after_secs` field is always `None` because the client doesn't parse the `Retry-After` header.

- **Partial SSE events are discarded**: If a connection drops mid-event (before `\n\n`), that partial data is lost. This is correct behavior for SSE but means the last few characters of a response might be missing.

- **CLI doesn't break on stream errors**: The CLI logs the error but continues processing, which means it might display partial output followed by a retry.

## Follow-up Questions

- [ ] How does the Agent determine the retry delay (what triggers `RetryTimeoutFired`), and is there exponential backoff at the agent level?
- [ ] Could the `ProxyLlmClient` be enhanced to parse the `Retry-After` header for rate-limited responses to enable smarter retry timing?

## Related Discoveries

- [[003-request-flow]] - Overall request flow through the proxy
- [[006-llm-service-model-resolution]] - How the server handles model resolution and provider errors
