# Discovery: How are streaming responses handled when errors occur mid-stream?

> Category: Follow-ups (Data Flow)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Loom handles mid-stream errors through a **first-class error event pattern** where errors are treated as regular stream events rather than exceptional conditions. When an error occurs during streaming (after HTTP headers have been sent), the system emits an `LlmEvent::Error` variant that flows through the same SSE channel as successful events. This ensures clients are always notified of failures even when the stream has already started.

The architecture follows a three-layer approach:
1. **Provider Layer**: Each LLM provider (Anthropic, OpenAI, Vertex) has its own stream parser that converts provider-specific errors into the unified `LlmEvent::Error` type
2. **Proxy Layer**: The server wraps `LlmEvent::Error` into `LlmStreamEvent::Error { message: String }` for wire transmission
3. **Client Layer**: Clients receive the error event and can handle it appropriately (retry, display, or propagate)

This design means the HTTP response always returns 200 OK for streaming endpoints (since headers are sent before the stream starts), and errors are communicated in-band as part of the event stream.

## Evidence

### Core Error Types

**`LlmEvent` (internal representation)** — `crates/loom-common-core/src/llm.rs:68-83`:
```rust
pub enum LlmEvent {
    TextDelta { content: String },
    ToolCallDelta { call_id: String, tool_name: String, arguments_fragment: String },
    Completed(LlmResponse),
    /// An error occurred during streaming.
    Error(LlmError),
}
```

**`LlmStreamEvent` (wire format)** — `crates/loom-server/src/llm_proxy.rs:45-67`:
```rust
pub enum LlmStreamEvent {
    TextDelta { content: String },
    ToolCallDelta { call_id: String, tool_name: String, arguments_fragment: String },
    ServerQuery(ServerQuery),
    Completed { response: LlmProxyResponse },
    Error { message: String },
}
```

### Error Sources During Streaming

**1. HTTP/Network Errors** — All provider streams handle `reqwest::Error`:
```rust
// crates/loom-server-llm-anthropic/src/stream.rs:167-170
Poll::Ready(Some(Err(e))) => {
    error!(error = %e, "Stream error");
    *this.finished = true;
    return Poll::Ready(Some(LlmEvent::Error(LlmError::Http(e.to_string()))));
}
```

**2. Invalid UTF-8 Data**:
```rust
// crates/loom-server-llm-openai/src/stream.rs:86-91
Err(e) => {
    warn!(error = %e, "Invalid UTF-8 in stream");
    return Poll::Ready(Some(LlmEvent::Error(LlmError::InvalidResponse(format!(
        "Invalid UTF-8: {e}"
    )))));
}
```

**3. Provider API Errors** (sent as structured error events):
```rust
// crates/loom-server-llm-anthropic/src/stream.rs:332-339
StreamEvent::Error { error } => {
    error!(
        error_type = %error.error_type,
        message = %error.message,
        "Stream error from API"
    );
    Err(LlmError::Api(error.message))
}
```

**4. JSON Parse Errors** (OpenAI/Vertex check for error response format):
```rust
// crates/loom-server-llm-openai/src/stream.rs:202-209
Err(e) => {
    if let Ok(error_response) = serde_json::from_str::<OpenAIError>(data) {
        warn!(error_type = ?error_response.error.error_type, "OpenAI API error in stream");
        return Some(LlmEvent::Error(LlmError::Api(error_response.error.message)));
    }
    warn!(error = %e, data = data, "Failed to parse stream chunk");
}
```

### Server-Side Error Forwarding

The proxy converts `LlmEvent::Error` to SSE events — `crates/loom-server/src/llm_proxy.rs:374-386`:
```rust
LlmEvent::Error(err) => {
    tracing::warn!(error = %err, "stream error");
    let stream_event = LlmStreamEvent::Error {
        message: err.to_string(),
    };
    match serde_json::to_string(&stream_event) {
        Ok(json) => Event::default().event("llm").data(json),
        Err(e) => {
            tracing::error!(error = %e, "failed to serialize error event");
            continue;
        }
    }
}
```

### Client-Side Error Handling

**CLI** — `crates/loom-cli/src/main.rs:579-581`:
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
}
```

**ACP Agent** — `crates/loom-cli-acp/src/agent.rs:436-438`:
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
    return Err(AcpError::Llm(e));
}
```

**State Machine** — `crates/loom-common-core/src/agent.rs:214-253`:
```rust
// CallingLlm + LlmEvent::Error -> Error state with retry
(AgentState::CallingLlm { conversation, retries }, AgentEvent::LlmEvent(LlmEvent::Error(e))) => {
    let new_retries = *retries + 1;
    if new_retries < self.config.max_retries {
        self.state = AgentState::Error { conversation, error: AgentError::Llm(e), retries: new_retries, origin: ErrorOrigin::Llm };
        AgentAction::WaitForInput  // Will retry after timeout
    } else {
        self.state = AgentState::WaitingForUserInput { conversation };
        AgentAction::DisplayError(AgentError::Llm(e).to_string())
    }
}
```

### Stream Termination Behavior

| Error Type | Stream Continues? | `finished` Flag |
|------------|-------------------|-----------------|
| HTTP/Network Error | No | Set to `true` |
| Invalid UTF-8 | No | Set to `true` |
| Provider API Error | No | Set to `true` |
| JSON Parse Error | Yes (skipped) | Unchanged |
| Serialization Error | Yes (skipped) | Unchanged |

**Key Files:**
- `crates/loom-common-core/src/llm.rs` — Core `LlmEvent` and `LlmError` types
- `crates/loom-common-core/src/error.rs` — `LlmError` enum definition
- `crates/loom-server/src/llm_proxy.rs` — SSE response creation and error forwarding
- `crates/loom-server-llm-anthropic/src/stream.rs` — Anthropic stream parser with error handling
- `crates/loom-server-llm-openai/src/stream.rs` — OpenAI stream parser with error handling
- `crates/loom-server-llm-vertex/src/stream.rs` — Vertex stream parser with error handling
- `crates/loom-server-llm-proxy/src/stream.rs` — Client-side proxy stream parser
- `crates/loom-common-core/src/agent.rs` — State machine error handling with retry logic

## Implications

1. **Errors are in-band**: Since HTTP 200 is returned before streaming starts, all errors must be communicated as stream events. Clients cannot rely on HTTP status codes for streaming error detection.

2. **Graceful degradation**: Minor parse errors (malformed JSON chunks) are logged and skipped, allowing the stream to continue. Only fatal errors (network, API errors) terminate the stream.

3. **Retry integration**: The agent state machine automatically retries on `LlmEvent::Error` up to `max_retries` times, with exponential backoff via `RetryTimeoutFired` events.

4. **Provider normalization**: All three providers (Anthropic, OpenAI, Vertex) map their specific error formats to the unified `LlmError` type, making client code provider-agnostic.

5. **Client disconnection handling**: The server detects client disconnection when `tx.send()` fails and cleanly terminates the spawned task.

## Follow-up Questions

- [ ] How does the retry backoff timing work for streaming errors? (exponential backoff, jitter?)
- [ ] How does the feature flag SSE client handle reconnection after errors?

## Related Discoveries

- [[005-retry-mechanism]] — Retry mechanism for LLM errors
- [[002-state-machine-orchestration]] — State machine handles error transitions
- [[004-tool-execution-system]] — Tool execution also has error handling patterns
