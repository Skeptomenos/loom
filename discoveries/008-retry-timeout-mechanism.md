# Discovery: How does the Agent determine the retry delay (what triggers RetryTimeoutFired), and is there exponential backoff at the agent level?

> Category: Follow-up (Data Flow)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The Agent state machine in `loom-common-core` is **intentionally passive** regarding retry timing. It does **not** calculate delays or implement exponential backoff internally. Instead, it relies on an **external runtime/scheduler** to fire the `RetryTimeoutFired` event after an appropriate delay. This is a deliberate architectural decision that separates the state machine logic from I/O concerns.

When an LLM error occurs in the `CallingLlm` state and retries remain (< `max_retries`, default 3), the Agent transitions to the `Error` state and returns `AgentAction::WaitForInput`. This signals to the caller that it should:
1. Wait for an appropriate backoff period
2. Fire `AgentEvent::RetryTimeoutFired` to trigger the retry

**Critical finding**: Neither the CLI REPL (`loom-cli/src/main.rs`) nor the ACP agent (`loom-cli-acp/src/agent.rs`) currently implements this timer. The CLI simply logs `LlmEvent::Error` and continues, while the ACP agent returns an error immediately. This means **the Agent state machine's retry mechanism is currently unused in production code**.

The exponential backoff logic exists in `loom-common-http/src/retry.rs`, but this is used for low-level HTTP request retries (e.g., connection failures), not for application-level LLM request retries through the Agent state machine.

## Evidence

**Agent returns WaitForInput on error** (`crates/loom-common-core/src/agent.rs:229-242`):
```rust
if new_retries < self.config.max_retries {
    let conv = conversation.clone();
    self.state = AgentState::Error {
        conversation: conv,
        error: AgentError::Llm(e),
        retries: new_retries,
        origin: ErrorOrigin::Llm,
    };
    info!(
        from = old_state_name,
        to = "Error",
        "state transition (will retry)"
    );
    AgentAction::WaitForInput  // <-- Caller must schedule RetryTimeoutFired
}
```

**RetryTimeoutFired triggers retry** (`crates/loom-common-core/src/agent.rs:255-284`):
```rust
// Error + RetryTimeoutFired -> CallingLlm (retry)
(
    AgentState::Error {
        conversation,
        retries,
        origin: ErrorOrigin::Llm,
        ..
    },
    AgentEvent::RetryTimeoutFired,
) => {
    let request = LlmRequest { ... };
    self.state = AgentState::CallingLlm {
        conversation: conv,
        retries: new_retries,
    };
    AgentAction::SendLlmRequest(request)
}
```

**Spec confirms external timer responsibility** (`specs/error-handling.md:253-255`):
```markdown
### RetryTimeoutFired Event

External systems (runtime/scheduler) fire `AgentEvent::RetryTimeoutFired` after a backoff delay
```

**CLI does not implement retry timer** (`crates/loom-cli/src/main.rs:579-581`):
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
    // No retry scheduling - just logs and continues
}
```

**ACP agent returns error immediately** (`crates/loom-cli-acp/src/agent.rs:436-438`):
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
    return Err(AcpError::Llm(e));  // No retry - immediate failure
}
```

**HTTP-level backoff exists but is separate** (`crates/loom-common-http/src/retry.rs:66-78`):
```rust
fn calculate_delay(cfg: &RetryConfig, attempt: u32) -> Duration {
    let exponential_delay = cfg.base_delay.as_secs_f64() * cfg.backoff_factor.powi(attempt as i32);
    let capped_delay = exponential_delay.min(cfg.max_delay.as_secs_f64());
    let final_delay = if cfg.jitter {
        let jitter_factor = 0.5 + fastrand::f64();
        capped_delay * jitter_factor
    } else {
        capped_delay
    };
    Duration::from_secs_f64(final_delay)
}
```

**Key Files:**
- `crates/loom-common-core/src/agent.rs` - Agent state machine with Error state and RetryTimeoutFired handling
- `crates/loom-common-core/src/state.rs` - AgentEvent::RetryTimeoutFired definition
- `crates/loom-common-core/src/config.rs` - AgentConfig with max_retries (default: 3)
- `crates/loom-common-http/src/retry.rs` - HTTP-level exponential backoff (separate from Agent)
- `crates/loom-cli/src/main.rs` - CLI REPL (does not use Agent state machine)
- `crates/loom-cli-acp/src/agent.rs` - ACP agent (does not implement retry timer)
- `specs/error-handling.md` - Specification documenting the intended behavior

## Implications

- **The Agent state machine's retry mechanism is currently dead code**: While the state transitions are implemented and tested, no production code actually schedules the `RetryTimeoutFired` event. Transient LLM errors will either be logged and ignored (CLI) or cause immediate failure (ACP).

- **Two-tier retry architecture**: Loom has HTTP-level retries (connection failures, 429/503 responses) via `loom-common-http::retry`, and application-level retries via the Agent state machine. Only the HTTP-level retries are currently functional.

- **Implementing Agent-level retries would require**: Modifying the CLI/ACP to detect when the Agent enters the `Error` state, calculate a backoff delay (potentially reusing `loom-common-http::calculate_delay`), sleep for that duration, and then send `AgentEvent::RetryTimeoutFired`.

- **The separation is intentional**: Keeping timing logic out of the state machine makes it easier to test (no async/timers in unit tests) and allows different runtimes to implement different backoff strategies.

## Follow-up Questions

- [ ] Should the CLI and ACP agent be updated to implement the retry timer, or is the current behavior (immediate failure/logging) intentional?
- [ ] Would a shared "Agent runtime" abstraction be useful to encapsulate the retry timer logic for reuse across CLI, ACP, and future consumers?

## Related Discoveries

- [[007-sse-error-handling]] - How SSE errors propagate to the Agent
- [[003-request-flow]] - Overall request flow through the system
