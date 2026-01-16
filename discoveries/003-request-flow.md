# Discovery: What is the complete request flow from user input through the LLM proxy to tool execution and back?

> Category: Follow-up (from Architecture)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The complete request flow in Loom follows a **6-phase cycle** that moves data from user input through a server-side LLM proxy to tool execution and back. The architecture is designed around a key security principle: **API keys never leave the server**. The CLI acts as the "hands" (executing tools locally), while the server acts as the "brain" (routing LLM requests with stored credentials).

### Phase 1: User Input Capture (CLI)
User input enters via `tokio::io::stdin()` in the `run_repl` function (`crates/loom-cli/src/main.rs:491`). The input is trimmed and wrapped in a `Message::user(input)` object, then appended to the conversation history.

### Phase 2: LLM Request Construction
An `LlmRequest` is built containing:
- The full conversation history (`messages`)
- Available tool definitions (`tools`)
- Model identifier (e.g., "default" which maps to `claude-3-5-sonnet-20241022`)

### Phase 3: Proxy Communication (CLI → Server)
The `ProxyLlmClient` sends the request via HTTP POST to `/proxy/{provider}/stream` (e.g., `/proxy/anthropic/stream`). The client attaches a bearer token for authentication if configured.

### Phase 4: Server-Side LLM Call (Server → Provider)
The server's `llm_proxy.rs` handlers receive the request and:
1. Retrieve `LlmService` from application state
2. Verify the provider is configured (`has_anthropic()`, `has_openai()`)
3. Forward to the actual provider client (e.g., `AnthropicClient`)
4. The provider returns an `LlmStream` of `LlmEvent`s

### Phase 5: SSE Streaming (Provider → Server → CLI)
The server wraps each `LlmEvent` into an `LlmStreamEvent`, serializes to JSON, and sends as SSE with `event: llm` header. The CLI's `ProxyLlmStream` parses these SSE events back into `LlmEvent`s. Text deltas are printed immediately to stdout.

### Phase 6: Tool Execution & Loop (CLI)
When `LlmEvent::Completed` contains tool calls:
1. Tools are executed **sequentially** via `execute_tool()` (lines 612-656)
2. Each tool result is wrapped in `Message::tool(call_id, tool_name, result)`
3. Results are appended to conversation history
4. If mutating tools ran (`edit_file`, `bash`), the `PostToolsHook` triggers auto-commit
5. A new `LlmRequest` is sent with tool results, looping back to Phase 2

## Evidence

### CLI REPL Loop (Phase 1-2)
```rust
// crates/loom-cli/src/main.rs:532-542
let user_message = Message::user(input);
messages.push(user_message.clone());

let request = loom_common_core::LlmRequest::new("default")
    .with_messages(messages.clone())
    .with_tools(tool_definitions.to_vec());
```

### ProxyLlmClient (Phase 3)
```rust
// crates/loom-server-llm-proxy/src/client.rs:108-114
fn stream_url(&self) -> String {
    format!(
        "{}/proxy/{}/stream",
        self.base_url.trim_end_matches('/'),
        self.provider
    )
}
```

### Server-Side Handler (Phase 4)
```rust
// crates/loom-server/src/llm_proxy.rs:155-159
let stream = service
    .complete_streaming_anthropic(request)
    .await
    .map_err(map_llm_error)?;
Ok(create_sse_response(stream))
```

### SSE Event Construction (Phase 5)
```rust
// crates/loom-server/src/llm_proxy.rs:329-337
LlmEvent::TextDelta { content } => {
    let stream_event = LlmStreamEvent::TextDelta { content };
    match serde_json::to_string(&stream_event) {
        Ok(json) => Event::default().event("llm").data(json),
        // ...
    }
}
```

### Tool Execution Loop (Phase 6)
```rust
// crates/loom-cli/src/main.rs:612-656
for tool_call in &tool_calls {
    let outcome = execute_tool(tool_registry, tool_call, tool_ctx).await;
    // ... format result as Message::tool() ...
    messages.push(Message::tool(&tool_call.id, &tool_call.tool_name, &tool_result));
}
```

**Key Files:**
- `crates/loom-cli/src/main.rs` — REPL loop, tool execution, message construction
- `crates/loom-server-llm-proxy/src/client.rs` — `ProxyLlmClient` implementation
- `crates/loom-server-llm-proxy/src/stream.rs` — SSE parsing on client side
- `crates/loom-server/src/llm_proxy.rs` — Server-side proxy handlers
- `crates/loom-server-llm-service/src/service.rs` — `LlmService` provider routing
- `specs/architecture.md` — Design documentation

## Implications

When working in this codebase:

- **Adding new LLM providers**: Only requires server-side changes. Add a new client in `loom-llm-{provider}`, register in `LlmService`, and add routes in `llm_proxy.rs`. Clients automatically get access via the existing `ProxyLlmClient`.

- **Debugging LLM issues**: Check both client-side (`ProxyLlmClient` logs) and server-side (`llm_proxy.rs` logs). The SSE format means you can inspect raw HTTP traffic to see the exact events.

- **Tool execution is synchronous**: Despite the state machine supporting parallel execution, the CLI currently executes tools sequentially. This is intentional for simpler debugging and avoiding race conditions in file operations.

- **The "double-wrap" SSE strategy**: The server wraps provider events into `LlmStreamEvent`, allowing it to inject custom events (like `ServerQuery` for human-in-the-loop) without breaking the provider's protocol.

- **Auto-commit integration**: The `PostToolsHook` state is the integration point. After mutating tools complete, this hook runs before returning results to the LLM.

## Follow-up Questions

- [ ] How does the LlmService handle model resolution (e.g., mapping "default" to specific model versions) and what happens when a model is not available?
- [ ] How does the SSE error handling work when the connection drops mid-stream, and how does the client recover?

## Related Discoveries

- [[001-primary-purpose]] — Establishes Loom as an AI coding agent with tool execution capabilities
- [[002-concurrent-tool-execution]] — Details the state machine design and why tools are executed sequentially
