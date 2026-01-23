# Design Patterns

This document catalogs recurring design patterns and idioms for building robust software.

---

## State Machine Pattern

**Use Case**: Complex workflows with explicit states and transitions.

The state machine separates state logic from async I/O:
- **State**: Current status (Idle, Processing, Waiting, Error, etc.)
- **Event**: Input that triggers transitions
- **Action**: Output the caller must execute

```rust
// The state machine is PASSIVE - it doesn't drive itself
fn handle_event(&mut self, event: Event) -> Action {
    match (&self.state, event) {
        (State::Idle, Event::Start(input)) => {
            self.state = State::Processing { data: input };
            Action::BeginWork(input)
        }
        (State::Processing { .. }, Event::Complete(result)) => {
            self.state = State::Idle;
            Action::ReturnResult(result)
        }
        // ... more transitions
    }
}
```

**Why**: 
- State logic is synchronous and testable
- Caller handles async operations
- Explicit transitions prevent invalid states
- Easy to visualize and reason about

**Testing**: Generate random event sequences and verify invariants hold.

---

## Registry Pattern

**Use Case**: Dynamic lookup of implementations by name/key.

```rust
pub struct Registry<T> {
    items: HashMap<String, Box<T>>,
}

impl<T> Registry<T> {
    pub fn register(&mut self, name: &str, item: Box<T>) {
        self.items.insert(name.to_string(), item);
    }
    
    pub fn get(&self, name: &str) -> Option<&T> {
        self.items.get(name).map(|t| t.as_ref())
    }
    
    pub fn list(&self) -> Vec<&str> {
        self.items.keys().map(|s| s.as_str()).collect()
    }
}
```

**Why**:
- Enables dynamic discovery and invocation by name
- Decouples registration from usage
- Supports plugin architectures

**Example Uses**: Tool registries, command handlers, middleware chains.

---

## Proxy Pattern

**Use Case**: Intermediary that controls access to another service.

```
Client (LocalProxy) → HTTP → Server (ProxyService) → External API
```

```rust
// Client side - appears to call API directly
impl ApiClient for ProxyClient {
    async fn request(&self, req: Request) -> Result<Response, Error> {
        self.http_client
            .post(&self.proxy_url)
            .json(&req)
            .send()
            .await?
            .json()
            .await
    }
}
```

**Why**:
- API keys never leave the server
- Centralized credential management
- Rate limiting and caching at proxy layer
- Request/response transformation

---

## SSE Streaming Pattern

**Use Case**: Real-time streaming responses over HTTP.

Server-Sent Events with structured payloads:

```rust
// Server wraps events in SSE format
let event = StreamEvent::Delta { content };
Event::default()
    .event("message")
    .data(serde_json::to_string(&event)?)

// Client parses SSE back to typed events
fn parse_sse_line(line: &str) -> Option<StreamEvent> {
    if line.starts_with("data: ") {
        let json = &line[6..];
        serde_json::from_str::<StreamEvent>(json).ok()
    } else {
        None
    }
}
```

**Why**:
- Low latency for streaming data
- Server can inject custom events (e.g., progress updates, queries)
- Works through proxies and load balancers
- Simple protocol (just text lines)

---

## Trait-Based Extensibility

**Use Case**: Adding new implementations without modifying core code.

Define abstractions as traits with sensible defaults:

```rust
#[async_trait]
pub trait Handler: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    
    // Default implementation
    fn metadata(&self) -> Metadata {
        Metadata {
            name: self.name().to_string(),
            description: self.description().to_string(),
        }
    }
    
    async fn handle(&self, input: Input, ctx: &Context) -> Result<Output>;
}
```

**Why**:
- New implementations don't touch core code
- Clear contracts via trait requirements
- Default methods reduce boilerplate
- Easy to mock for testing

---

## Builder Pattern

**Use Case**: Constructing objects with many optional parameters.

```rust
let request = Request::new("endpoint")
    .with_method(Method::POST)
    .with_body(body)
    .with_timeout(Duration::from_secs(30))
    .with_headers(headers);
```

**Implementation**:
```rust
impl Request {
    pub fn new(endpoint: &str) -> Self {
        Self {
            endpoint: endpoint.to_string(),
            method: Method::GET,
            body: None,
            timeout: None,
            headers: HashMap::new(),
        }
    }
    
    pub fn with_method(mut self, method: Method) -> Self {
        self.method = method;
        self
    }
    
    pub fn with_body(mut self, body: impl Into<Body>) -> Self {
        self.body = Some(body.into());
        self
    }
    
    // ... more builders
}
```

**Why**:
- Avoids large constructors with many optional parameters
- Self-documenting at call site
- Compile-time validation of required fields

---

## Error Response Pattern

**Use Case**: Consistent error format across HTTP APIs.

```rust
pub struct ErrorResponse {
    pub error: String,      // Machine-readable code: "not_found", "validation_failed"
    pub message: String,    // Human-readable explanation
}

// Helper functions for common cases
pub fn not_found(message: impl Into<String>) -> (StatusCode, Json<ErrorResponse>) {
    (StatusCode::NOT_FOUND, Json(ErrorResponse {
        error: "not_found".to_string(),
        message: message.into(),
    }))
}

pub fn bad_request(message: impl Into<String>) -> (StatusCode, Json<ErrorResponse>) {
    (StatusCode::BAD_REQUEST, Json(ErrorResponse {
        error: "bad_request".to_string(),
        message: message.into(),
    }))
}
```

**Why**:
- Consistent format across all endpoints
- Machine-readable codes enable programmatic handling
- Human-readable messages for debugging/display

---

## Secret Wrapper Pattern

**Use Case**: Preventing accidental exposure of sensitive values.

```rust
pub struct Secret<T>(T);

impl<T> Secret<T> {
    pub fn new(value: T) -> Self {
        Self(value)
    }
    
    pub fn expose(&self) -> &T {
        &self.0
    }
}

impl<T> Debug for Secret<T> {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}

impl<T> Display for Secret<T> {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}
```

**Why**:
- Secrets automatically redacted in logs, debug output, serialization
- Explicit `.expose()` call required to access value
- Compile-time safety via type system

**Usage**:
```rust
let api_key = Secret::new("sk-12345".to_string());
debug!(key = ?api_key, "using key");  // Logs "[REDACTED]"
client.set_auth(api_key.expose());     // Explicit access
```

---

## Hook Pattern

**Use Case**: Extensible post-processing without modifying core logic.

```rust
#[async_trait]
pub trait PostActionHook: Send + Sync {
    async fn run(&self, context: &HookContext) -> Result<HookResult>;
}

pub enum HookResult {
    Continue,           // Proceed normally
    Abort(String),      // Stop with error
    Retry,              // Retry the action
}

// Example: Auto-save hook
impl PostActionHook for AutoSaveHook {
    async fn run(&self, ctx: &HookContext) -> Result<HookResult> {
        if ctx.has_modifications() {
            self.save_service.save().await?;
        }
        Ok(HookResult::Continue)
    }
}
```

**Why**:
- Decouples post-processing from core logic
- Hooks can be composed and ordered
- Easy to add/remove behaviors

---

## Retry with Backoff Pattern

**Use Case**: Handling transient failures in network operations.

```rust
fn calculate_delay(config: &RetryConfig, attempt: u32) -> Duration {
    let base = config.base_delay.as_secs_f64();
    let exponential = base * config.backoff_factor.powi(attempt as i32);
    let capped = exponential.min(config.max_delay.as_secs_f64());
    
    if config.jitter {
        let jitter = rand::thread_rng().gen_range(0.5..1.5);
        Duration::from_secs_f64(capped * jitter)
    } else {
        Duration::from_secs_f64(capped)
    }
}
```

**Why**:
- Exponential growth prevents overwhelming recovering services
- Jitter prevents thundering herd (synchronized retries)
- Cap prevents excessive delays

See [retry-strategy.md](retry-strategy.md) for full details.

---

## Typed Router Pattern

**Use Case**: Compile-time separation of route categories.

```rust
pub struct PublicRouter(Router);   // No auth required
pub struct AuthedRouter(Router);   // Auth required

impl PublicRouter {
    pub fn new() -> Self {
        Self(Router::new())
    }
    
    pub fn route(mut self, path: &str, handler: impl Handler) -> Self {
        self.0 = self.0.route(path, handler);
        self
    }
}

impl AuthedRouter {
    pub fn new() -> Self {
        Self(Router::new().layer(auth_layer()))
    }
    
    // Same route method...
}

// Usage
let public = PublicRouter::new()
    .route("/health", get(health_handler));

let protected = AuthedRouter::new()
    .route("/api/users", get(list_users));
```

**Why**:
- Compile-time separation of public vs protected routes
- Prevents accidentally exposing authenticated endpoints
- Clear intent at definition site

---

## Summary

| Pattern | Purpose | Key Benefit |
|---------|---------|-------------|
| State Machine | Complex workflows | Testable, explicit transitions |
| Registry | Dynamic lookup | Discovery by name |
| Proxy | Access control | Server-side credentials |
| SSE Streaming | Real-time responses | Low latency |
| Trait Extensibility | Adding implementations | No core modifications |
| Builder | Object construction | Ergonomic optional params |
| Error Response | HTTP errors | Consistent, machine-readable |
| Secret Wrapper | Credential protection | Auto-redaction |
| Hook | Post-processing | Composable behavior |
| Retry with Backoff | Transient failures | Thundering herd prevention |
| Typed Router | Route organization | Auth requirement clarity |
