# Discovery: How does the LLM proxy pattern work in detail? (ProxyLlmClient, SSE streaming)

> Category: Follow-up (Architecture)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The LLM proxy pattern is a **server-side gateway architecture** that keeps API keys secure on the server while providing a unified streaming interface to clients. The pattern consists of three layers:

1. **Client Layer (`ProxyLlmClient`)**: Implements the `LlmClient` trait by making HTTP requests to the server's proxy endpoints instead of calling LLM providers directly. It transforms the server's SSE response back into `LlmEvent`s using `ProxyLlmStream`.

2. **Server Proxy Layer (`llm_proxy.rs`)**: Receives requests from clients, delegates to the `LlmService`, and transforms provider-specific `LlmEvent`s into a unified wire format (`LlmStreamEvent`) sent as SSE with `event: llm` headers.

3. **Provider Layer (`LlmService` + provider clients)**: Holds API keys securely, makes actual calls to LLM providers (Anthropic, OpenAI, Vertex), and parses provider-specific SSE formats into the common `LlmEvent` enum.

The key insight is the **double-wrap strategy**: provider SSE → `LlmEvent` → `LlmStreamEvent` (wire format) → client parses back to `LlmEvent`. This allows the server to inject custom events (like `ServerQuery` for human-in-the-loop) without breaking provider protocols.

## Evidence

### ProxyLlmClient Structure (`crates/loom-server-llm-proxy/src/client.rs`)

```rust
pub struct ProxyLlmClient {
    base_url: String,
    provider: LlmProvider,
    http_client: reqwest::Client,
    auth_token: Option<SecretString>,
}

impl ProxyLlmClient {
    fn complete_url(&self) -> String {
        format!("{}/proxy/{}/complete", self.base_url.trim_end_matches('/'), self.provider)
    }

    fn stream_url(&self) -> String {
        format!("{}/proxy/{}/stream", self.base_url.trim_end_matches('/'), self.provider)
    }
}

#[async_trait]
impl LlmClient for ProxyLlmClient {
    async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError> {
        let response = self.http_client.post(&self.stream_url()).json(&request).send().await?;
        let byte_stream = response.bytes_stream();
        let proxy_stream = ProxyLlmStream::new(Box::pin(byte_stream));
        Ok(LlmStream::new(Box::pin(proxy_stream)))
    }
}
```

### SSE Stream Parser (`crates/loom-server-llm-proxy/src/stream.rs`)

```rust
pub struct ProxyLlmStream {
    inner: Pin<Box<dyn Stream<Item = Result<Bytes, reqwest::Error>> + Send>>,
    buffer: String,
}

impl Stream for ProxyLlmStream {
    type Item = LlmEvent;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        // Buffer incoming bytes until \n\n delimiter found
        // Parse "event: llm\ndata: {...}" format
        // Deserialize JSON into LlmStreamEvent, convert to LlmEvent
    }
}
```

### Server-Side SSE Creation (`crates/loom-server/src/llm_proxy.rs`)

```rust
fn create_sse_response(stream: LlmStream) -> Sse<impl futures::Stream<Item = Result<Event, Infallible>>> {
    let (tx, rx) = tokio::sync::mpsc::channel::<Result<Event, Infallible>>(32);

    tokio::spawn(async move {
        while let Some(event) = stream.next().await {
            let sse_event = match event {
                LlmEvent::TextDelta { content } => {
                    let stream_event = LlmStreamEvent::TextDelta { content };
                    Event::default().event("llm").data(serde_json::to_string(&stream_event)?)
                }
                // ... other variants
            };
            tx.send(Ok(sse_event)).await.ok();
        }
    });

    Sse::new(ReceiverStream::new(rx))
}
```

### Wire Format (`LlmStreamEvent`)

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum LlmStreamEvent {
    TextDelta { content: String },
    ToolCallDelta { call_id: String, tool_name: String, arguments_fragment: String },
    ServerQuery(ServerQuery),  // For human-in-the-loop
    Completed { response: LlmProxyResponse },
    Error { message: String },
}
```

### Route Registration (`crates/loom-server/src/api.rs`)

```rust
"/proxy/anthropic/complete" -> post(llm_proxy::proxy_anthropic_complete)
"/proxy/anthropic/stream"   -> post(llm_proxy::proxy_anthropic_stream)
"/proxy/openai/complete"    -> post(llm_proxy::proxy_openai_complete)
"/proxy/openai/stream"      -> post(llm_proxy::proxy_openai_stream)
"/proxy/vertex/complete"    -> post(llm_proxy::proxy_vertex_complete)
"/proxy/vertex/stream"      -> post(llm_proxy::proxy_vertex_stream)
```

**Key Files:**
- `crates/loom-server-llm-proxy/src/client.rs` — ProxyLlmClient implementation
- `crates/loom-server-llm-proxy/src/stream.rs` — SSE parser (ProxyLlmStream)
- `crates/loom-server/src/llm_proxy.rs` — Server-side proxy handlers and SSE creation
- `crates/loom-server-llm-service/src/service.rs` — LlmService that routes to provider clients
- `crates/loom-common-core/src/llm.rs` — Core LlmClient trait and LlmStream type

## Implications

- **Security**: API keys never leave the server. Clients only need a server URL and optional auth token.
- **Adding new providers**: Only requires server-side changes. Add a new `loom-llm-{provider}` crate, register in `LlmService`, add routes in `llm_proxy.rs`. Clients automatically get access via existing `ProxyLlmClient`.
- **Model resolution**: The server substitutes `"default"` model with provider-specific defaults (e.g., `claude-opus-4-20250514` for Anthropic).
- **Error handling**: Pre-stream errors (connection, auth) return HTTP error codes. Mid-stream errors are sent as `LlmStreamEvent::Error` events.
- **Extensibility**: The `ServerQuery` event type enables future human-in-the-loop features without breaking the streaming protocol.

## Follow-up Questions

- [ ] How does the thread system persist conversations and integrate with the proxy?
- [ ] How do provider-specific stream parsers (e.g., AnthropicStream) handle partial tool calls and state accumulation?

## Related Discoveries

- [[003-crate-relationships]] — Explains the overall crate architecture including the proxy pattern
- [[012-streaming-error-handling]] — Details error handling during streaming
