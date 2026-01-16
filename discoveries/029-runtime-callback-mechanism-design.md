# Discovery: Should the runtime be generic over the progress/callback mechanism, or use a fixed trait like `AgentRuntimeCallbacks`?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Use a fixed trait (`AgentRuntimeCallbacks`) stored as a trait object (`Arc<dyn AgentRuntimeCallbacks>`), NOT a generic type parameter.**

This recommendation is based on the established pattern in the Loom codebase, where callback/hook mechanisms consistently use:
1. A fixed trait definition with `Send + Sync + 'static` bounds
2. Trait object storage (`Arc<dyn Trait>`) in the consuming struct
3. A builder method that accepts a generic and converts to trait object

## Evidence

### Pattern 1: AnalyticsHook in loom-flags

The `FlagsClient` uses this exact pattern for analytics callbacks:

```rust
// crates/loom-flags/src/analytics.rs:152-170
#[async_trait]
pub trait AnalyticsHook: Send + Sync + 'static {
    async fn on_flag_evaluated(&self, exposure: FlagExposure);
}

pub type SharedAnalyticsHook = Arc<dyn AnalyticsHook>;

pub struct NoOpAnalyticsHook;

#[async_trait]
impl AnalyticsHook for NoOpAnalyticsHook {
    async fn on_flag_evaluated(&self, _exposure: FlagExposure) {}
}
```

```rust
// crates/loom-flags/src/client.rs:216-224
pub struct FlagsClient {
    // ... other fields ...
    analytics_hook: SharedAnalyticsHook,  // Trait object, NOT generic
}

// Builder accepts generic, converts to trait object
pub fn analytics_hook<H: AnalyticsHook>(mut self, hook: H) -> Self {
    self.analytics_hook = Some(Arc::new(hook));
    self
}
```

### Pattern 2: MergeAuditHook in loom-server-analytics

```rust
// crates/loom-server-analytics/src/identity_resolution.rs:37-57
pub trait MergeAuditHook: Send + Sync {
    fn on_merge(&self, details: PersonMergeDetails);
}

pub struct NoOpMergeAuditHook;

impl MergeAuditHook for NoOpMergeAuditHook {
    fn on_merge(&self, _details: PersonMergeDetails) {}
}

pub type SharedMergeAuditHook = Arc<dyn MergeAuditHook>;

// The service stores an Option<SharedMergeAuditHook>
pub struct IdentityResolutionService<R: AnalyticsRepository> {
    repository: R,
    audit_hook: Option<SharedMergeAuditHook>,  // Trait object
}
```

### Pattern 3: ServerQueryHandler in loom-common-core

```rust
// crates/loom-common-core/src/server_query.rs:177-188
#[async_trait]
pub trait ServerQueryHandler: Send + Sync {
    async fn handle_query(&self, query: ServerQuery)
        -> Result<ServerQueryResponse, ServerQueryError>;
}
```

### When Generics ARE Used

The codebase uses generics when:
1. **Testability requires swapping implementations** (e.g., `AutoCommitService<G: GitClient, L: LlmClient>`)
2. **The type is a core dependency, not a callback** (e.g., `IdentityResolutionService<R: AnalyticsRepository>`)
3. **Performance is critical and monomorphization helps** (rare in this codebase)

Callbacks/hooks are NOT in this category - they're optional, fire-and-forget notifications.

## Design Rationale

### Why Fixed Trait + Trait Object?

| Aspect | Fixed Trait + Trait Object | Generic Type Parameter |
|--------|---------------------------|------------------------|
| **API Simplicity** | `AgentRuntime` has no type params | `AgentRuntime<C: Callbacks>` everywhere |
| **Composition** | Easy to store in collections | Requires `Box<dyn>` anyway for heterogeneous |
| **Default Behavior** | `NoOpCallbacks` as default | Must specify type even for no-op |
| **Runtime Flexibility** | Can swap at runtime | Fixed at compile time |
| **Compile Times** | Single instantiation | Monomorphization per callback type |
| **Codebase Consistency** | Matches AnalyticsHook, MergeAuditHook | Would be an outlier |

### Why NOT Generic?

1. **Callbacks are not hot paths**: Unlike `LlmClient::complete()` which is called frequently with large data, callbacks are lightweight notifications. The vtable overhead is negligible.

2. **Avoids "generic explosion"**: If `AgentRuntime<C>` were generic, every function that uses it would need to be generic too, or use `AgentRuntime<dyn AgentRuntimeCallbacks>` anyway.

3. **Simplifies testing**: Tests can use `NoOpCallbacks` without specifying types.

4. **Matches existing patterns**: The codebase already uses this pattern for `AnalyticsHook`, `MergeAuditHook`, and `PostToolHook`.

## Recommended Implementation

```rust
// crates/loom-agent-runtime/src/callbacks.rs

use async_trait::async_trait;
use loom_common_core::ToolProgress;
use std::sync::Arc;
use std::time::Duration;

/// Callbacks for receiving progress updates from the agent runtime.
///
/// All methods have default no-op implementations, allowing consumers
/// to only implement the callbacks they care about.
#[async_trait]
pub trait AgentRuntimeCallbacks: Send + Sync + 'static {
    /// Called when the LLM emits a text chunk.
    async fn on_llm_text_delta(&self, _content: &str) {}
    
    /// Called when a tool execution starts.
    async fn on_tool_started(&self, _call_id: &str, _tool_name: &str) {}
    
    /// Called when a tool reports progress.
    async fn on_tool_progress(&self, _call_id: &str, _progress: &ToolProgress) {}
    
    /// Called when a tool execution completes.
    async fn on_tool_completed(&self, _call_id: &str, _tool_name: &str, _success: bool) {}
    
    /// Called when a retry is scheduled after a rate limit.
    async fn on_retry_scheduled(&self, _attempt: u32, _delay: Duration) {}
    
    /// Called when an error occurs.
    async fn on_error(&self, _error: &str) {}
}

/// Shared reference to callbacks.
pub type SharedCallbacks = Arc<dyn AgentRuntimeCallbacks>;

/// No-op callbacks for testing or when progress reporting isn't needed.
#[derive(Debug, Clone, Copy, Default)]
pub struct NoOpCallbacks;

#[async_trait]
impl AgentRuntimeCallbacks for NoOpCallbacks {}
```

```rust
// crates/loom-agent-runtime/src/runtime.rs

pub struct AgentRuntime {
    agent: Agent,
    llm_client: Arc<dyn LlmClient>,
    tool_registry: Arc<ToolRegistry>,
    retry_config: RetryConfig,
    callbacks: SharedCallbacks,  // Trait object, NOT generic
}

impl AgentRuntime {
    pub fn builder() -> AgentRuntimeBuilder {
        AgentRuntimeBuilder::default()
    }
}

pub struct AgentRuntimeBuilder {
    // ... fields ...
    callbacks: Option<SharedCallbacks>,
}

impl AgentRuntimeBuilder {
    /// Sets the callbacks for progress reporting.
    ///
    /// Accepts any type implementing `AgentRuntimeCallbacks`.
    pub fn callbacks<C: AgentRuntimeCallbacks>(mut self, callbacks: C) -> Self {
        self.callbacks = Some(Arc::new(callbacks));
        self
    }
    
    pub fn build(self) -> Result<AgentRuntime, BuildError> {
        let callbacks = self.callbacks.unwrap_or_else(|| Arc::new(NoOpCallbacks));
        // ...
    }
}
```

## Implications

1. **The runtime struct has no type parameters**: This simplifies usage across CLI, TUI, and ACP.

2. **Builder pattern for configuration**: Matches `FlagsClient::builder()` pattern.

3. **Default no-op behavior**: Consumers don't need to provide callbacks if they don't care about progress.

4. **Async callbacks**: Allows non-blocking operations (channel sends, stdout flushes).

5. **Single trait, not split**: Keep all callbacks in one trait for simplicity. Splitting (e.g., `LlmCallbacks`, `ToolCallbacks`) adds complexity without clear benefit - most consumers want all or nothing.

## Follow-up Questions

- [ ] Should the callbacks trait include a `on_state_changed(&self, old: &AgentState, new: &AgentState)` method for debugging/logging?
- [ ] Should there be a `CallbacksBuilder` for composing multiple callback implementations (e.g., logging + analytics)?

## Related Discoveries

- [[028-agent-runtime-trait-bounds]] — Recommended `Send + Sync` bounds and async callbacks
- [[016-shared-agent-runtime-analysis]] — Proposes the AgentRuntime abstraction
- [[008-retry-timeout-mechanism]] — Documents the passive state machine design
