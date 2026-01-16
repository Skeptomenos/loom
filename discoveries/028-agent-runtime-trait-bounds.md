# Discovery: What trait bounds should `AgentRuntime` require for maximum flexibility while maintaining type safety?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Based on analysis of the existing codebase patterns, the `AgentRuntime` should use **trait objects with `Send + Sync` bounds** for its core dependencies, following the established conventions in Loom. However, it should be **generic over a callbacks trait** to allow different consumers (CLI, TUI, ACP) to receive progress updates in their preferred format.

**Recommended trait bounds:**

```rust
pub struct AgentRuntime {
    agent: Agent,
    llm_client: Arc<dyn LlmClient>,           // Send + Sync via trait definition
    tool_registry: Arc<ToolRegistry>,          // Send + Sync (contains Box<dyn Tool>)
    retry_config: RetryConfig,
    post_tool_hooks: Vec<Arc<dyn PostToolHook>>,
}

#[async_trait]
pub trait AgentRuntimeCallbacks: Send + Sync {
    async fn on_llm_text_delta(&self, content: &str);
    async fn on_tool_started(&self, call_id: &str, tool_name: &str);
    async fn on_tool_progress(&self, call_id: &str, progress: &ToolProgress);
    async fn on_tool_completed(&self, call_id: &str, tool_name: &str, success: bool);
    async fn on_retry_scheduled(&self, attempt: u32, delay: Duration);
    async fn on_error(&self, error: &str);
}
```

This design:
1. Uses `Arc<dyn Trait>` for dependencies (matches `LlmClient`, `Tool`, store patterns)
2. Requires `Send + Sync` on all traits (standard across 40+ traits in codebase)
3. Uses `#[async_trait]` for async methods (consistent with all async traits)
4. Keeps the runtime itself non-generic for simpler usage

## Evidence

**Codebase convention: All async traits require `Send + Sync`** (grep found 43 matches):
```rust
// crates/loom-common-core/src/llm.rs:140
pub trait LlmClient: Send + Sync {
    async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;
    async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError>;
}

// crates/loom-cli-tools/src/registry.rs:9
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    async fn invoke(&self, args: Value, ctx: &ToolContext) -> Result<Value, ToolError>;
}

// crates/loom-server-jobs/src/job.rs:10
pub trait Job: Send + Sync {
    async fn execute(&self, ctx: &JobContext) -> Result<(), JobError>;
}
```

**Existing progress reporting pattern** (`crates/loom-common-core/src/state.rs:14-21`):
```rust
pub struct ToolProgress {
    pub fraction: Option<f32>,
    pub message: Option<String>,
    pub units_processed: Option<u64>,
}
```

**TUI uses event-driven updates** (`crates/loom-tui-component/src/lib.rs:45-63`):
```rust
pub trait Component: Send + Sync {
    fn handle_event(&mut self, event: &Event) -> Vec<Action>;
    fn update(&mut self, action: &Action) -> Vec<Action>;
    fn render(&self, frame: &mut Frame, area: Rect, ctx: &RenderContext);
}
```

**Retry logic already exists and is reusable** (`crates/loom-common-http/src/retry.rs:66-78`):
```rust
fn calculate_delay(cfg: &RetryConfig, attempt: u32) -> Duration {
    let exponential_delay = cfg.base_delay.as_secs_f64() * cfg.backoff_factor.powi(attempt as i32);
    // ... jitter logic
}
```

**Key Files:**
- `crates/loom-common-core/src/llm.rs` — `LlmClient` trait with `Send + Sync` bounds
- `crates/loom-cli-tools/src/registry.rs` — `Tool` trait with `Send + Sync` bounds
- `crates/loom-common-core/src/state.rs` — `ToolProgress`, `AgentEvent`, `AgentAction` types
- `crates/loom-common-http/src/retry.rs` — Reusable `RetryConfig` and `calculate_delay`
- `crates/loom-tui-component/src/lib.rs` — `Component` trait pattern for UI updates

## Design Rationale

### Why `Arc<dyn Trait>` over generics?

1. **Consistency**: The codebase already uses `Arc<dyn LlmClient>` in `Agent::new()` and `Box<dyn Tool>` in `ToolRegistry`. Generics would be a departure from established patterns.

2. **Simplicity**: Trait objects avoid the "generic explosion" problem where `AgentRuntime<L, T, H>` would require specifying types at every call site.

3. **Runtime flexibility**: Allows swapping implementations (e.g., mock LLM for testing) without recompilation.

### Why a separate `AgentRuntimeCallbacks` trait?

1. **Decoupling**: The runtime shouldn't know about CLI stdout, TUI widgets, or ACP message channels. Callbacks abstract this.

2. **Testability**: Tests can use a `NoopCallbacks` implementation that ignores all events.

3. **Flexibility**: Different consumers need different things:
   - CLI: Print to stdout, show spinners
   - TUI: Update `ToolPanel` widget state
   - ACP: Send SSE events to connected clients

### Why async callbacks?

1. **Non-blocking**: CLI might need to flush stdout, TUI might need to acquire a lock, ACP might need to send over a channel.

2. **Consistency**: All other async traits in the codebase use `#[async_trait]`.

## Proposed Implementation

```rust
// crates/loom-agent-runtime/src/lib.rs

use async_trait::async_trait;
use loom_common_core::{Agent, AgentAction, AgentEvent, LlmClient, ToolProgress};
use loom_common_http::RetryConfig;
use loom_cli_tools::ToolRegistry;
use std::sync::Arc;
use std::time::Duration;

#[async_trait]
pub trait AgentRuntimeCallbacks: Send + Sync {
    async fn on_llm_text_delta(&self, content: &str) {}
    async fn on_tool_started(&self, call_id: &str, tool_name: &str) {}
    async fn on_tool_progress(&self, call_id: &str, progress: &ToolProgress) {}
    async fn on_tool_completed(&self, call_id: &str, tool_name: &str, success: bool) {}
    async fn on_retry_scheduled(&self, attempt: u32, delay: Duration) {}
    async fn on_error(&self, error: &str) {}
}

/// No-op callbacks for testing or when progress reporting isn't needed.
pub struct NoopCallbacks;

#[async_trait]
impl AgentRuntimeCallbacks for NoopCallbacks {}

pub struct AgentRuntime {
    agent: Agent,
    llm_client: Arc<dyn LlmClient>,
    tool_registry: Arc<ToolRegistry>,
    retry_config: RetryConfig,
}

impl AgentRuntime {
    pub fn new(
        agent: Agent,
        llm_client: Arc<dyn LlmClient>,
        tool_registry: Arc<ToolRegistry>,
    ) -> Self {
        Self {
            agent,
            llm_client,
            tool_registry,
            retry_config: RetryConfig::default(),
        }
    }

    pub fn with_retry_config(mut self, config: RetryConfig) -> Self {
        self.retry_config = config;
        self
    }

    pub async fn run(
        &mut self,
        input: Message,
        callbacks: &dyn AgentRuntimeCallbacks,
    ) -> Result<(), AgentRuntimeError> {
        // Implementation drives the Agent state machine
        // Fires callbacks at appropriate points
        // Handles retry logic via RetryTimeoutFired events
        todo!()
    }
}
```

## Implications

- **The callbacks trait should have default implementations**: This allows consumers to only implement the callbacks they care about. A CLI might only implement `on_llm_text_delta` and `on_error`.

- **Consider a `CancellationToken` parameter**: The current `run_prompt_loop` in ACP checks `session.is_cancelled()`. The runtime should accept a `tokio_util::sync::CancellationToken` for graceful shutdown.

- **Post-tool hooks should be part of the runtime**: The `PostToolHook` pattern from `loom-cli-auto-commit` should be integrated, with hooks stored as `Vec<Arc<dyn PostToolHook>>`.

- **The runtime should NOT be generic over `LlmClient`**: Using `Arc<dyn LlmClient>` matches the existing `Agent::new()` signature and avoids breaking changes.

## Follow-up Questions

- [ ] Should `AgentRuntimeCallbacks` be split into separate traits (e.g., `LlmCallbacks`, `ToolCallbacks`) for finer-grained implementation?
- [ ] Should the runtime accept a `CancellationToken` for graceful shutdown, or use a different cancellation mechanism?

## Related Discoveries

- [[016-shared-agent-runtime-analysis]] — Proposes the AgentRuntime abstraction
- [[008-retry-timeout-mechanism]] — Documents the passive state machine design
- [[011-read-only-tools-parallelization]] — Parallel execution would be easier with shared runtime
